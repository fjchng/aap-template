# 6_Deployment

## Project Information

- **Project Name**: [Project Name]
- **Version**: [Version Number]
- **Date**: [Date]
- **Author**: [Author Name]

---

## Deployment Strategy

### Environment Pipeline

- **Development**: Feature development and testing
- **Staging**: Pre-production testing and QA validation
- **Production**: Live application serving real users

### Deployment Models

#### Blue-Green Deployment

- Deploy to green environment → Test → Switch traffic → Keep blue for rollback
- **Benefits**: Zero downtime, immediate rollback
- **Use for**: Production releases with high availability requirements

#### Rolling Updates

- Replace instances one by one with health checks
- **Benefits**: Resource efficient, gradual risk exposure
- **Use for**: Standard deployments with automatic rollback

#### Canary Releases

- Route small percentage to new version → Monitor → Gradually increase
- **Benefits**: Risk mitigation, real user feedback
- **Use for**: Major feature releases requiring validation

---

## Infrastructure as Code

### Docker Configuration

```dockerfile
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY src/ ./src/
RUN npm run build

FROM node:18-alpine AS production
RUN addgroup -g 1001 -S nodejs && adduser -S nextjs -u 1001 -G nodejs
WORKDIR /app
COPY --from=builder --chown=nextjs:nodejs /app/dist ./dist
COPY --from=builder --chown=nextjs:nodejs /app/node_modules ./node_modules

HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1

USER nextjs
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

### Docker Compose (Development)

```yaml
version: "3.8"
services:
  app:
    build: .
    ports: ["3000:3000"]
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://postgres:password@db:5432/myapp_dev
    depends_on: [db, redis]

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=myapp_dev
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    ports: ["5432:5432"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp-backend
  template:
    metadata:
      labels:
        app: myapp-backend
    spec:
      containers:
        - name: backend
          image: myapp/backend:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: app-secrets
                  key: database-url
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 5
```

---

## CI/CD Pipeline

### GitHub Actions Workflow

```yaml
name: Build and Deploy

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "18"
          cache: "npm"
      - run: npm ci
      - run: npm run lint
      - run: npm run test
      - run: npm run build

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to staging
        run: |
          # Build and push Docker image
          docker build -t myapp:staging .
          docker push registry/myapp:staging

          # Deploy to staging environment
          kubectl set image deployment/myapp-staging backend=registry/myapp:staging
          kubectl rollout status deployment/myapp-staging

  deploy-production:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to production
        run: |
          # Build and push Docker image
          docker build -t myapp:production .
          docker push registry/myapp:production

          # Deploy to production environment
          kubectl set image deployment/myapp-production backend=registry/myapp:production
          kubectl rollout status deployment/myapp-production
```

### Database Migrations

```typescript
// Migration script example
export class InitialSchema implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.createTable(
      new Table({
        name: "users",
        columns: [
          {
            name: "id",
            type: "uuid",
            isPrimary: true,
            generationStrategy: "uuid",
          },
          { name: "email", type: "varchar", isUnique: true },
          { name: "password_hash", type: "varchar" },
          {
            name: "created_at",
            type: "timestamp",
            default: "CURRENT_TIMESTAMP",
          },
        ],
      })
    );
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.dropTable("users");
  }
}
```

---

## Monitoring and Health Checks

### Health Check Endpoints

```typescript
// routes/health.ts
router.get("/health", (req, res) => {
  res.json({
    status: "healthy",
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION,
    uptime: process.uptime(),
  });
});

router.get("/ready", async (req, res) => {
  try {
    await DatabaseService.healthCheck();
    res.json({ status: "ready" });
  } catch (error) {
    res.status(503).json({ status: "not ready", error: error.message });
  }
});
```

### Basic Monitoring

```typescript
// middleware/monitoring.ts
export const requestMetrics = (req, res, next) => {
  const startTime = Date.now();

  res.on("finish", () => {
    const duration = Date.now() - startTime;
    console.log({
      method: req.method,
      url: req.url,
      status: res.statusCode,
      duration,
      timestamp: new Date().toISOString(),
    });
  });

  next();
};
```

---

## Security

### Environment Configuration

```typescript
// config/environment.ts
const requiredEnvVars = ["DATABASE_URL", "JWT_SECRET", "EMAIL_API_KEY"];

// Validate required environment variables
const missing = requiredEnvVars.filter((envVar) => !process.env[envVar]);
if (missing.length > 0) {
  throw new Error(
    `Missing required environment variables: ${missing.join(", ")}`
  );
}

export const config = {
  app: {
    port: parseInt(process.env.PORT || "3000"),
    environment: process.env.NODE_ENV || "development",
  },
  database: {
    url: process.env.DATABASE_URL,
    ssl: process.env.NODE_ENV === "production",
  },
  auth: {
    jwtSecret: process.env.JWT_SECRET,
    jwtExpiresIn: process.env.JWT_EXPIRES_IN || "15m",
  },
};
```

### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database-url: <base64-encoded-database-url>
  jwt-secret: <base64-encoded-jwt-secret>
  email-api-key: <base64-encoded-email-api-key>
```

---

## Backup and Recovery

### Database Backup Script

```bash
#!/bin/bash
# scripts/backup-database.sh

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="backup_${TIMESTAMP}.sql"

# Create backup
pg_dump $DATABASE_URL --format=custom --compress=9 > $BACKUP_FILE

# Upload to cloud storage
aws s3 cp $BACKUP_FILE s3://myapp-backups/database/

# Clean up local file
rm $BACKUP_FILE

echo "Backup completed: $BACKUP_FILE"
```

### Automated Backup (Kubernetes CronJob)

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-backup
spec:
  schedule: "0 2 * * *" # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:15-alpine
              command: ["/bin/bash", "-c"]
              args:
                - |
                  pg_dump $DATABASE_URL --format=custom > /backup/backup_$(date +%Y%m%d_%H%M%S).sql
                  aws s3 cp /backup/backup_*.sql s3://$BACKUP_BUCKET/database/
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: app-secrets
                      key: database-url
          restartPolicy: OnFailure
```

---

## Deployment Checklists

### Pre-Deployment Checklist

- [ ] All tests passing
- [ ] Code review approved
- [ ] Security scan completed
- [ ] Database migrations tested
- [ ] Environment variables configured
- [ ] Rollback plan prepared
- [ ] Stakeholder approval received

### Post-Deployment Checklist

- [ ] Health checks passing
- [ ] Application responding correctly
- [ ] Database migrations applied
- [ ] Key user flows tested
- [ ] Monitoring metrics normal
- [ ] No increase in error rates

---

## Emergency Procedures

### Rollback Process

```bash
# Kubernetes rollback
kubectl rollout undo deployment/myapp-backend

# Verify rollback
kubectl rollout status deployment/myapp-backend

# Check application health
curl -f https://api.myapp.com/health
```

### Common Issues and Solutions

#### High Error Rate

```bash
# Check recent logs
kubectl logs -f deployment/myapp-backend --tail=100

# Scale up replicas
kubectl scale deployment/myapp-backend --replicas=5

# Rollback if needed
kubectl rollout undo deployment/myapp-backend
```

#### Database Issues

```bash
# Check database connectivity
pg_isready -h $DB_HOST -U $DB_USER

# Check active connections
echo "SELECT count(*) FROM pg_stat_activity;" | psql $DATABASE_URL
```

---

**Document Status**: [Draft/Review/Approved]  
**Last Updated**: [Date]  
**Next Review**: [Date]
