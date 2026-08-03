---
title: "Two Kinds of Caller, One Gateway: Extensible Token Auth for Multi-Tenant APIs"
published: false
description: How to authenticate first-party app users and third-party API partners through one Spring Boot gateway — a single identity contract, per-caller token issuers, a pluggable verification SPI, revocable JWTs, and scope-as-URI-prefix authorization at the edge.
tags: springboot, java, security, api
canonical_url: https://aayushmiglani.github.io/posts/multi-identity-token-auth-gateway/
cover_image:
---

> This article is also available on my blog: [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/multi-identity-token-auth-gateway/)

## The problem: a second kind of caller shows up

Picture a backend that started life with exactly one audience: people using your app. They sign in with a phone number and an OTP, or an email and a password, and from then on every request carries a token that says *who* they are. Your gateway verifies the token, figures out the user, and lets the request through. Simple, and it works for years.

Then the shape of the business changes. A partner company wants to call your API directly — server to server, no human in the loop — to read a slice of data or trigger a workflow on behalf of their own product. They don't have a phone number. They don't log in with an OTP. They authenticate with a client ID and a secret, the way machines do. And crucially, they must **not** be able to call your whole API: they get a narrow, explicit list of endpoints and nothing else.

The lazy way to absorb this is to reach into the existing token code and start branching. `if (isPartner) { ... } else { ... }` in the token issuer. Another branch in the verifier. A third in the gateway. Six months later a third caller type appears — an internal service, say — and now every one of those branches becomes a three-way `switch`. The auth layer, the most security-sensitive code you own, turns into a thicket of conditionals that no one wants to touch.

There is a cleaner shape. The trick is to notice that issuing a token and verifying a token are two different problems with two different amounts of variation — and to design each accordingly.

> **What you'll build.** A multi-caller auth layer for a Spring Boot platform: a single `Identity` contract every caller collapses into, a separate token *issuer* per caller type (because minting genuinely differs), one pluggable verification SPI that new caller types extend without edits, a token store that makes JWTs revocable, and a gateway filter chain that authorizes partners down to an explicit list of paths — deny-by-default at the edge.

## The mental model in 60 seconds

Three ideas carry the whole design. Hold them and the rest is detail.

1. **One identity contract.** However a caller authenticated and whatever token shape they carry, verification produces the same small value object: a subject (who), a set of scopes (what they're allowed to touch), and a bag of attributes (caller-specific config). Downstream code never asks "is this a partner?" — it just reads an `Identity`.
2. **Issuance varies, so keep issuers separate; verification converges, so unify it.** Minting a human session (refresh tokens, OTP, long expiry) looks nothing like minting a machine-to-machine token (no refresh, short expiry, client-secret login). Don't force them into one method. But every token, once minted, is signed the same way and verified through one entry point.
3. **Authorize at the edge.** The gateway authenticates the token, turns it into an `Identity`, and then — for callers that carry scopes — checks the requested path against that scope list before the request ever reaches a backend service.

```
  first-party caller                third-party partner
  (phone + OTP / pwd)               (client id + secret)
        │                                   │
        ▼                                   ▼
  ┌───────────────┐                 ┌───────────────────┐
  │ SessionIssuer │                 │  PartnerIssuer    │     ── issuance: separate
  │ access+refresh│                 │  short-lived,     │        per caller type
  │ token         │                 │  carries scopes   │
  └───────┬───────┘                 └─────────┬─────────┘
          │  signed JWT                       │  signed JWT
          └─────────────────┬─────────────────┘
                            ▼
                  ┌───────────────────┐
   every request  │      Gateway      │
   ─────────────▶ │                   │
                  │  AuthFilter ──▶ verifyToken()  ── unified verification
                  │     │              │              dispatch by token "type"
                  │     │              ▼
                  │     │        ┌──────────────┐
                  │     │        │   Identity   │  subject + scopes + attributes
                  │     │        └──────┬───────┘
                  │     ▼               │
                  │  ScopeFilter ◀──────┘    ── authorize: path vs scope list
                  │     │  (partners only; first-party = allow-all)
                  └─────┼─────────────────┘
                        ▼
                  backend services
```

## Phase 1 — One identity contract for every caller

Start at the end. Whatever a request's token says, the rest of your system should receive the same thing. Define it as a small, immutable value object:

```java
public record Identity(
        String subject,                  // who: user id, partner id, service name
        Set<String> scopes,              // what they may call (empty = unrestricted)
        Map<String, String> attributes   // caller-specific config, e.g. a tenant key
) {
    public boolean isScoped() {
        return scopes != null && !scopes.isEmpty();
    }
}
```

This single type is doing the heavy lifting. "Partner-ness" is not a flag or a subclass — it's just an `Identity` that happens to carry a non-empty `scopes` set. A first-party user is an `Identity` with no scopes (which, as we'll see, the gateway reads as "allowed everywhere"). An internal service you add next year is an `Identity` with its own scopes. Every consumer downstream — controllers, audit logging, rate limiters — speaks one language.

The `attributes` map is the quiet workhorse. A partner often comes with per-tenant configuration: a routing key, a limit override, a feature toggle. Carrying that on the identity means a downstream handler can read it without knowing — or caring — that it came from a partner token.

> **Keep this object dumb.** No behavior, no database handles, no Spring magic — just data. The whole point is that it's trivially constructable by any verifier and trivially readable by anything downstream. Resist the urge to hang "convenience" methods that hit a service off it.

## Phase 2 — Separate issuers, because minting really is different

It is tempting to make one `createToken(...)` that takes a caller type and branches. Don't. The two flows differ in ways that aren't incidental:

- **First-party sessions** need a long-lived *refresh* token alongside the access token, because a human session should survive for days without re-login. They're minted after an interactive login (OTP, password).
- **Partner tokens** have *no* refresh token. A machine re-authenticates with its client secret whenever it needs a fresh token — cheap, and it avoids handing a long-lived credential to a server you don't control. They're minted after a client-credentials check, and they carry scopes.

So you keep the original session issuer untouched and add a second issuer next to it. The session issuer stays roughly as it always was:

```java
@Service
@RequiredArgsConstructor
public class SessionTokenIssuer {

    private final TokenSigner signer;          // shared signing (see Phase 3)
    private final TokenStore store;            // shared store    (see Phase 4)

    public TokenPair issueForUser(String userId, ClientChannel channel) {
        long ttl = channel == ClientChannel.WEB ? webTtl : appTtl;

        String access  = signer.sign(builder -> builder
                .withSubject(userId)
                .withExpiresAt(Instant.now().plusMillis(ttl)));
        String refresh = signer.sign(builder -> builder
                .withSubject(userId)
                .withExpiresAt(Instant.now().plusMillis(refreshTtl)));

        store.put(accessKey(userId), access, ttl);
        store.put(refreshKey(userId), refresh, refreshTtl);
        return new TokenPair(access, refresh);
    }
}
```

The partner issuer is a different method on a different class. It authenticates the client secret, loads the scopes and attributes provisioned for that client, and bakes them into a single short-lived token:

```java
@Service
@RequiredArgsConstructor
public class PartnerTokenIssuer {

    private final TokenSigner signer;
    private final TokenStore store;
    private final PasswordEncoder passwordEncoder;
    private final PartnerCredentialRepository credentials;

    public String authenticateAndIssue(String clientId, String clientSecret) {
        PartnerCredential cred = credentials.findActiveByClientId(clientId)
                .orElseThrow(() -> new AuthException("invalid_client"));

        // Compare against a stored hash — never the raw secret.
        if (!passwordEncoder.matches(clientSecret, cred.secretHash())) {
            throw new AuthException("invalid_client");
        }

        String token = signer.sign(builder -> builder
                .withSubject(cred.partnerId())
                .withClaim("type", "PARTNER")               // the discriminator
                .withArrayClaim("scopes", cred.scopes())     // baked in at mint time
                .withClaim("attributes", cred.attributes())
                .withExpiresAt(Instant.now().plusMillis(partnerTtl)));

        store.put(partnerKey(cred.partnerId()), token, partnerTtl);
        return token;
    }
}
```

Notice two things. First, the partner's permissions are *baked into the token at mint time* — the scopes come from the credential store once, at issuance, not on every request. That's a deliberate trade-off: fewer database reads on the hot path, at the cost of a partner's permissions being frozen for the lifetime of a token. Second, the partner token carries a `type` claim. The session token does not. That asymmetry is the hinge of the next phase.

## Phase 3 — One verification SPI, open for extension

Verification is where you *want* convergence, because every caller's token has to be checked by the same gateway on the same hot path. This is the part you make extensible. Define a strategy interface — a service provider interface (SPI) — for "things that can turn a decoded token into an `Identity`":

```java
public interface TokenVerifier {

    /** Does this verifier handle tokens of the given type? */
    boolean supports(String tokenType);

    /** Validate the token's claims/state and produce the caller's identity. */
    Identity verify(DecodedToken decoded, String rawToken);
}
```

Now the central `TokenService` doesn't know about partners, users, or services. It verifies the signature once, reads the `type` claim, and dispatches to whichever verifier claims it. Spring injects *every* bean implementing the interface as a list, so adding a verifier is adding a class — nothing here changes:

```java
@Service
@RequiredArgsConstructor
public class TokenService {

    private final TokenSigner signer;
    private final List<TokenVerifier> verifiers;   // all implementations, injected

    public Identity verifyToken(String rawToken) {
        DecodedToken decoded = signer.verifySignature(rawToken);   // throws if invalid/expired
        String type = decoded.claim("type");                       // null for legacy sessions

        return verifiers.stream()
                .filter(v -> v.supports(type))
                .findFirst()
                .orElseThrow(() -> new AuthException("unsupported_token_type"))
                .verify(decoded, rawToken);
    }
}
```

The two implementations are small and self-contained:

```java
@Service
class PartnerTokenVerifier implements TokenVerifier {
    private final TokenStore store;

    public boolean supports(String type) { return "PARTNER".equals(type); }

    public Identity verify(DecodedToken decoded, String rawToken) {
        String partnerId = decoded.subject();
        assertCurrent(store, partnerKey(partnerId), rawToken);     // Phase 4
        return new Identity(
                partnerId,
                Set.copyOf(decoded.listClaim("scopes")),
                decoded.mapClaim("attributes"));
    }
}

@Service
class SessionTokenVerifier implements TokenVerifier {
    private final TokenStore store;

    // Legacy session tokens predate the type system — they have NO type claim.
    public boolean supports(String type) { return type == null; }

    public Identity verify(DecodedToken decoded, String rawToken) {
        String userId = decoded.subject();
        assertCurrent(store, accessKey(userId), rawToken);
        return new Identity(userId, Set.of(), Map.of());   // no scopes = unrestricted
    }
}
```

This is the open/closed principle doing real work. Want to onboard internal service tokens next quarter? Mint them with `type=SERVICE` and write a `ServiceTokenVerifier`. The dispatcher, the gateway, and the `Identity` contract are all untouched. The most dangerous file in the codebase — the one that decides who gets in — stops growing.

> ⚠️ **The honest part: "no type claim" means first-party.** In a real system this design usually arrives *after* first-party tokens already exist in the wild. Those tokens were minted before anyone imagined a `type` claim, so they don't have one. Re-minting every live session to add `type=USER` would log everyone out. So instead, the session verifier claims tokens where `type == null`. It's a backward-compatibility seam, not elegance — but it's the correct call: you extend the model for new callers without invalidating tokens already in people's pockets. If you're building greenfield, give every token an explicit type from day one and skip the null.

### One signer for all of them

The reason verification can be unified at all is that every token, regardless of issuer, is signed by one component with one key. Centralize it so the algorithm lives in exactly one place:

```java
@Component
public class TokenSigner {

    @Value("${auth.rsa-signing-enabled:false}")
    private boolean rsaEnabled;

    private Algorithm algorithm() {
        // A feature flag lets you migrate symmetric → asymmetric with no code change.
        return rsaEnabled
                ? Algorithm.RSA256(publicKey, privateKey)   // asymmetric: verifiers need only the public key
                : Algorithm.HMAC256(sharedSecret);           // symmetric: one shared secret
    }
}
```

Hiding the algorithm behind a flag is a small thing that pays off once: when you outgrow a shared HMAC secret and want asymmetric keys (so downstream services can verify with a public key they can't forge with), it's a config change, not a migration. Load the keys once at startup and treat them as immutable — the signer becomes a stateless singleton that's safe to share across every request thread.

## Phase 4 — Make the JWT revocable with a token store

A pure JWT is stateless and unrevocable: anyone holding a validly-signed, unexpired token is in, and there is no "log out" short of waiting for expiry. For a lot of systems that's unacceptable — you need to be able to kill a session, and you often want only one active token per subject. The fix is to make the JWT *stateful*: store the current token in a fast key/value store (Redis is the usual choice), keyed by subject, and require the presented token to match it.

```java
static void assertCurrent(TokenStore store, String key, String rawToken) {
    String current = store.get(key);
    if (current == null || !current.equals(rawToken)) {
        // signature was valid, but this token is no longer the active one
        throw new AuthException("token_revoked");
    }
}
```

That one equality check buys you three behaviors for free:

- **Logout works.** Delete the key; the next request with that token fails.
- **Single active session.** Issuing a new token overwrites the stored value, so the previous token instantly stops verifying — a fresh login elsewhere boots the old one.
- **Stolen-but-stale tokens die.** Even a valid signature isn't enough if the token isn't the current one.

Set the store entry's TTL equal to the JWT's expiry so the two can never disagree about when a token dies. The cost is real and worth naming: every request now does a store lookup, so your gateway can't validate purely in-process — it trades the headline advantage of stateless JWTs for operational control. For most consumer and partner APIs, being able to revoke is worth a single Redis read.

> **Pick your point on the spectrum.** Fully stateless JWTs scale beautifully and revoke terribly. Store-backed tokens revoke instantly and cost a lookup. A middle path — short access-token TTL plus a revocation check only on refresh — splits the difference. There's no universally right answer; decide based on how badly you need instant logout.

## Phase 5 — Authorize at the edge with scopes as path prefixes

Authentication tells you *who* is calling. Authorization decides *what* they may call. For partners, that's the whole ballgame: they must reach only the endpoints you've granted them. The cleanest place to enforce it is the gateway, before the request touches any backend — and the simplest model that works is to make **scopes literally be URI prefixes**.

A partner provisioned with scopes `["/v1/orders", "/v1/catalog"]` may call `/v1/orders`, `/v1/orders/42`, and anything under `/v1/catalog/…` — and nothing else. No scope registry, no mapping table, no annotations to keep in sync. The grant list *is* the path set. The gateway runs two filters in order.

### 5a. The authentication filter — token to identity

```java
@Component
@Order(2)
public class AuthenticationFilter extends OncePerRequestFilter {

    private final AuthClient authClient;   // calls TokenService.verifyToken()

    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ... {
        if (isPublic(req.getRequestURI())) { chain.doFilter(req, res); return; }

        String token = req.getHeader("x-token");
        if (token == null) { res.sendError(401); return; }

        Identity identity;
        try { identity = authClient.verify(token); }
        catch (AuthException e) { res.sendError(401); return; }

        req.setAttribute("subject", identity.subject());
        if (identity.isScoped()) {
            req.setAttribute("scopes", identity.scopes());        // only partners get this
        }
        // Spread caller-specific config so any downstream handler can read it.
        identity.attributes().forEach(req::setAttribute);

        chain.doFilter(req, res);
    }
}
```

The key move: `scopes` lands on the request *only* when the identity is scoped. First-party users have empty scopes, so the attribute is never set for them — which is exactly what makes the next filter a no-op for humans.

### 5b. The scope filter — path against grant list

```java
@Component
@Order(3)
public class ScopeFilter extends OncePerRequestFilter {

    @SuppressWarnings("unchecked")
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ... {
        Set<String> scopes = (Set<String>) req.getAttribute("scopes");

        // No scopes on the request = unscoped caller (a first-party user). Allow through.
        if (scopes == null || scopes.isEmpty()) { chain.doFilter(req, res); return; }

        String uri = req.getRequestURI();
        boolean allowed = scopes.stream()
                .anyMatch(scope -> uri.equals(scope) || uri.startsWith(scope + "/"));

        if (!allowed) { res.sendError(403, "scope not permitted"); return; }
        chain.doFilter(req, res);
    }
}
```

Read the policy off those two filters: **deny-by-default for partners, allow-all for first-party users.** A partner can reach exactly the prefixes on its grant list and gets a 403 otherwise; a user (no scopes) sails through scope-checking and is constrained only by per-resource ownership rules deeper in the stack. That asymmetry is intentional — your own users are trusted to use your product; external callers are trusted with precisely what you wrote down.

> ⚠️ **The trap: `startsWith` is coarser than it looks.** Prefix matching is wonderfully simple and has two sharp edges. First, always compare against `scope + "/"` (or an exact match), never a bare `startsWith(scope)` — otherwise a grant for `/v1/order` silently also permits `/v1/orders-admin`. Second, a path prefix can't distinguish HTTP verbs: a scope of `/v1/orders` grants `GET` *and* `POST` *and* `DELETE`. If a partner should read but not write, prefixes alone won't express it — fold the method into the scope (`GET:/v1/orders`) or move to a real policy model. Know which one you need before you ship.

## Where to take this next

The design above is the load-bearing 90%. Three directions extend it cleanly:

1. **Add a third caller type to prove the seam.** Internal service-to-service tokens (`type=SERVICE`) are the natural next tenant. If adding them is "write one `TokenVerifier`, mint with a new type, provision scopes" and nothing in `TokenService` or the gateway changes, your SPI is doing its job. If you find yourself editing the dispatcher, the abstraction leaked — fix that first.
2. **Make scopes expressive without abandoning simplicity.** Method-aware scopes (`GET:/v1/orders`) cover most "read-only partner" needs with a one-line change to the matcher. Only reach for a full policy engine (OPA, Cedar, Spring Authorization Server scopes) when prefix-or-method genuinely can't model your grants — premature policy frameworks are their own kind of tar pit.
3. **Solve scope staleness if it bites.** Baking scopes into the token means revoking a partner's access means waiting for expiry (or nuking their store entry). If you need instant permission changes, keep the token thin (subject + type only) and look scopes up from the credential store per request — paying a lookup to buy freshness, the same trade you made in Phase 4.

---

## Recap

Letting a second kind of caller into an auth system built for one doesn't require forking your auth code — it requires separating the part that varies (issuance) from the part that should converge (verification), with one identity contract in the middle. The recipe:

1. Define one immutable `Identity` (subject + scopes + attributes) that every caller collapses into.
2. Keep a *separate issuer per caller type* — sessions get refresh tokens and interactive login; partners get short-lived, scoped, client-secret tokens. Don't merge them.
3. Sign every token with one centralized signer (algorithm behind a flag, keys loaded once).
4. Verify through one `TokenService` that dispatches by a `type` claim to a `List<TokenVerifier>` — adding a caller type is adding a class.
5. Use "no type claim" as the legacy first-party discriminator so you never have to re-mint live tokens.
6. Make tokens revocable by storing the current token per subject and requiring a match on verify.
7. Authorize partners at the gateway with scopes-as-URI-prefixes: deny-by-default for scoped callers, allow-all for unscoped users.

Ship that, and the next caller type — and the one after — is a small, boring, low-risk change to the riskiest code you own. Which is exactly what you want auth changes to be.

---

*Originally published at [aayushmiglani.github.io](https://aayushmiglani.github.io/posts/multi-identity-token-auth-gateway/). Found this useful? The [correlation-ID post](https://aayushmiglani.github.io/posts/correlation-id-spring-boot/) works the same servlet-filter seam for observability.*
