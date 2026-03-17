Architecture & LLD - Solution 1
ALB -> ECS Fargate NGINX (proxy only) -> private S3 + private API Gateway
Prepared for internal design review

Ranking for this option: Good. Assumption: internal-only Angular SPA for corporate users over Direct Connect/VPN, with a private REST API behind API Gateway.

