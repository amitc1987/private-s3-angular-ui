Architecture & LLD - Solution 1
ALB -> ECS Fargate NGINX (proxy only) -> private S3 + private API Gateway
Prepared for internal design review

Ranking for this option: Good. Assumption: internal-only Angular SPA for corporate users over Direct Connect/VPN, with a private REST API behind API Gateway.

1. Executive summary
NGINX is only a reverse proxy. It fetches UI objects from a private S3 bucket on demand and proxies /api/* to a private API Gateway.
Why it sits at this rank: Works with a hard 'UI must stay in S3' requirement and avoids browser CORS because the browser only sees one origin. The tradeoff is an extra hop for every static asset and more cache/header tuning than a local-file web tier.


2. Low-level design
Browser reaches the internal ALB over the corporate network and resolves one internal hostname, for example portal.company.internal.
ALB forwards all UI traffic to the ECS target group running NGINX.
NGINX proxies /api/* to the private API Gateway hostname and proxies all other paths to the private S3 bucket endpoint.
For missing UI object paths, NGINX internally serves /index.html so Angular can resolve the route client-side.

3. Design scenarios covered

CORS - No browser CORS issue if the Angular app calls only relative paths such as /api/orders and NGINX uses proxy_pass rather than HTTP redirects. CORS returns only if the browser is redirected to S3 or execute-api hostnames.
SPA routing - Covered. NGINX can proxy the requested object to S3, intercept a missing-object response, and internally return /index.html so refreshes on routes like /users/123 still work.
Security boundary - Strong. The bucket remains private, API Gateway remains private, and the user-facing entry point is the internal ALB.
Performance - Good enough for internal apps, but worse than serving files locally because each static object adds a proxy hop to S3.
Operational fit - Best when S3 is mandatory and the web tier must not persist UI artifacts.

4. Pros and cons

Pros :
Meets the strongest S3-only artifact constraint without exposing S3 directly to browsers.
Single browser origin avoids CORS when Angular uses relative /api/* paths.
NGINX can handle SPA fallback by serving /index.html when an object path is missing.
Keeps S3 private; users never call S3 or execute-api hostnames directly.
Reasonable compromise when the team does not want CloudFront.

Cons:
Every UI request becomes browser -> ALB -> NGINX -> S3, which increases latency and moving parts.
NGINX becomes a critical data-plane dependency for all static assets.
Cache-control, content-type, and compression behavior must be tuned carefully.
ECS tasks need private S3 and execute-api connectivity plus IAM or bucket-policy alignment.
More operational overhead than using S3 directly behind the load balancer.

5. Key design decisions
Keep the browser on one internal origin.
Use NGINX only as a proxy and not as a file store.
Use private S3 and private execute-api connectivity from the tasks.
Accept one extra runtime hop for static assets.
