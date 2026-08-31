# Stockroom API

A warehouse inventory management REST API built with Python 3.12 and FastAPI. Manages categories, products, and stock movements backed by PostgreSQL.

## Environment Variables

| Variable       | Default                                                              | Description                          |
|----------------|----------------------------------------------------------------------|--------------------------------------|
| `DATABASE_URL` | `postgresql+asyncpg://stockroom:stockroom@localhost:5432/stockroom`  | Async PostgreSQL connection string   |
| `API_KEY`      | `demo-api-key`                                                       | Bearer token for all API requests    |

Copy `.env.example` to `.env` and adjust as needed.

## GitHub Actions secrets

The CI workflow (`.github/workflows/ci.yml`) pushes Docker images to Amazon ECR using
short-lived credentials obtained via OIDC — no long-lived AWS access keys are stored in
GitHub.

Configure the following in **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | What it is | How to find it |
| ------ | ---------- | -------------- |
| `AWS_ROLE_ARN` | ARN of the IAM role GitHub Actions assumes via OIDC to push to ECR | Create an IAM role in AWS with a trust policy for `token.actions.githubusercontent.com` (see below); the ARN looks like `arn:aws:iam::123456789012:role/github-actions-stockroom` |
| `ECR_REGISTRY` | ECR registry hostname | `<your-12-digit-account-id>.dkr.ecr.<region>.amazonaws.com` — visible in the AWS Console under **Elastic Container Registry → Private registry** |

### Setting up the IAM OIDC trust

A single IAM role is shared by both `stockroom-api` and `stockroom-frontend`. Set the
same `AWS_ROLE_ARN` secret value in both repositories.

1. In the AWS Console go to **IAM → Identity providers → Add provider**.
   - Provider type: **OpenID Connect**
   - Provider URL: `https://token.actions.githubusercontent.com`
   - Audience: `sts.amazonaws.com`
   *(Skip if this provider already exists in your account.)*

2. Create an IAM role with the following trust policy, replacing `<ACCOUNT_ID>` and
   `<ORG>` with your values. The wildcard on the `sub` condition covers all
   `stockroom-*` repos so you don't need to update it when adding new service repos:

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": {
           "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
         },
         "Action": "sts:AssumeRoleWithWebIdentity",
         "Condition": {
           "StringEquals": {
             "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
           },
           "StringLike": {
             "token.actions.githubusercontent.com:sub": "repo:<ORG>/stockroom-*:*"
           }
         }
       }
     ]
   }
   ```

3. Attach a permission policy to the role that allows ECR push. The wildcard on the
   resource covers both `stockroom-api` and `stockroom-frontend` (and any future
   service repos that follow the same naming convention):

   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": [
           "ecr:GetAuthorizationToken"
         ],
         "Resource": "*"
       },
       {
         "Effect": "Allow",
         "Action": [
           "ecr:BatchCheckLayerAvailability",
           "ecr:CompleteLayerUpload",
           "ecr:InitiateLayerUpload",
           "ecr:PutImage",
           "ecr:UploadLayerPart",
           "ecr:BatchGetImage",
           "ecr:GetDownloadUrlForLayer"
         ],
         "Resource": "arn:aws:ecr:<REGION>:<ACCOUNT_ID>:repository/stockroom-*"
       }
     ]
   }
   ```

---

## Releasing

Versioned releases are driven by Git tags. Pushing a semver tag triggers the CI
workflow to build a Docker image and push it to ECR tagged with both the version
and `:latest`:

```bash
git tag v1.2.0
git push origin v1.2.0
```

This produces `stockroom-api:v1.2.0` and `stockroom-api:latest` in ECR.

After tagging, update `api_image_tag` in
[stockroom-deployment/terraform/terraform.tfvars](../stockroom-deployment/terraform/terraform.tfvars)
and open a PR against `release/prod` to deploy it.

## Running Locally

### With Docker Compose (recommended)

```yaml
# docker-compose.yml (create alongside the project)
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_USER: stockroom
      POSTGRES_PASSWORD: stockroom
      POSTGRES_DB: stockroom
    ports:
      - "5432:5432"

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+asyncpg://stockroom:stockroom@db:5432/stockroom
    depends_on:
      - db
```

```bash
docker compose up --build
```

### Direct (uvicorn)

```bash
pip install -e .
cp .env.example .env
uvicorn app.main:app --reload
```

## Running Migrations

```bash
# Apply all migrations
DATABASE_URL=postgresql+asyncpg://stockroom:stockroom@localhost:5432/stockroom \
  alembic upgrade head

# Generate a new migration after model changes
alembic revision --autogenerate -m "describe your change"
```

## API Endpoints

All endpoints require the header `X-Api-Key: <your api key>`.

Interactive docs available at `http://localhost:8000/docs`.

### Categories — `/api/v1/categories`

| Method   | Path                      | Description          |
|----------|---------------------------|----------------------|
| `GET`    | `/api/v1/categories`      | List all categories  |
| `POST`   | `/api/v1/categories`      | Create a category    |
| `GET`    | `/api/v1/categories/{id}` | Get a category       |
| `PUT`    | `/api/v1/categories/{id}` | Update a category    |
| `DELETE` | `/api/v1/categories/{id}` | Delete a category    |

### Products — `/api/v1/products`

| Method   | Path                    | Description                                     |
|----------|-------------------------|-------------------------------------------------|
| `GET`    | `/api/v1/products`      | List all products (optional `?category_id=`)    |
| `POST`   | `/api/v1/products`      | Create a product                                |
| `GET`    | `/api/v1/products/{id}` | Get a product                                   |
| `PUT`    | `/api/v1/products/{id}` | Update a product                                |
| `DELETE` | `/api/v1/products/{id}` | Delete a product                                |

### Stock Movements — `/api/v1/stock-movements`

| Method | Path                      | Description                                                    |
|--------|---------------------------|----------------------------------------------------------------|
| `GET`  | `/api/v1/stock-movements` | List movements, ordered newest first (optional `?product_id=`) |
| `POST` | `/api/v1/stock-movements` | Record a movement; updates `quantity_in_stock`                 |

**POST body example:**

```json
{
  "product_id": "uuid-here",
  "movement_type": "in",
  "quantity": 50,
  "reason": "Initial stock receipt"
}
```

A `400` is returned if an `"out"` movement would push stock below zero.
