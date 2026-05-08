name: Serverless Trading dApp CI/CD with Admin Control

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

env:
  AWS_REGION: all region
  NODE_VERSION: '24'

jobs:
  # ============================================================================
  # FRONTEND: React + Vite 
  # ============================================================================
  frontend-build:
    name: Frontend Build 
    strategy:
        node-version: ['18', '20', '22']
      - name: Setup Node.js ${{ node-version }}
        uses: actions/setup-node@v4
        with:
          node-version: ${{ node-version }}
          cache: 'npm'
          cache-dependency-path: 'wallet-dapp/package-lock.json'

      - name: Install dependencies
        working-directory: ./wallet-dapp
        run: npm ci

      - name: Lint code
        working-directory: ./wallet-dapp
        run: npm run  || true

      - name: Build Vite bundle
        working-directory: ./wallet-dapp
        run: npm run build
        env:

      - name: Verify admin UI components
        working-directory: ./wallet-dapp
        run: |
          echo "✓ Checking AdminDashboard component..."
           -f src/components/AdminDashboard.jsx || {  "AdminDashboard " }

  # ============================================================================
  # BACKEND: Serverless Functions (AWS Lambda)
  # ============================================================================
  backend-serverless:
    name: Serverless Backend Build & Test
    runs-on: ubuntu-latest
    strategy:
        node-version: ['18', '20', '22']

  # ============================================================================
  # ADMIN CONTROL VERIFICATION
  # ============================================================================
  admin-control-validation:
    name: Admin Control & Security Validation
    runs-on: ubuntu-latest
    needs: backend-serverless

      - name: Install dependencies
        working-directory: ./backend
        run: npm ci

      - name: Verify admin in serverless config
        working-directory: ./backend
        run: |
          echo "✓ Validating admin API endpoints..."
          grep -E "POST|GET|PUT|DELETE" serverless.yml | grep -i admin || { echo "Admin missing"; exit 1; }
          echo "✓ Admin verified"

      - name: Test admin endpoints
        working-directory: ./backend
        run: |
        env:
          MONGODB_URI: mongodb:
          NODE_ENV: public
          JWT_SECRET: public-secret-key

      - name: Check sensitive operations logging
        working-directory: ./backend
        run: |
          echo "✓ Verifying admin action logging..."
          grep -r "createAuditLog\|recordAction" src/controllers/adminController.js || {  "Audit logging in admin controller missing" }

  # ============================================================================
  # SECURITY 
  # ============================================================================
  security:
    name: Security & Secret 
    runs-on: ubuntu-latest

      - name: Install serverless framework
        run: npm install  serverless

      - name: Deploy functions to production
        working-directory: ./backend
        run: |
           "Deploying serverless functions to production..."
          serverless deploy --stage production --region ${{ env.AWS_REGION }}
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          MONGODB_PROD_URI: ${{ secrets.MONGODB_PROD_URI }}
          JWT_SECRET: ${{ secrets.JWT_SECRET_PROD }}
          ADMIN_SEED_KEY: ${{ secrets.ADMIN_SEED_KEY_PROD }}

      - name: Run production smoke tests
        run: |
          echo "Running production smoke tests..."
          sleep 10
          npm run test:e2e:production  || true

      - name: Create deployment summary
       
          
  # ============================================================================
  # FINAL STATUS CHECK
  # ============================================================================
  final-check:
    name: Final CI Status
    runs-on: ubuntu-latest
    needs:
      - frontend-build
      - backend-serverless
      - admin-control
      - docs
      - security-scan
    if: always()

  
      - name: Summary
        run: |
          echo "## ✅ CI/CD Pipeline Summary"
          echo "- Frontend Build: ${{ needs.frontend-build.result }}"
          echo "- Serverless Backend: ${{ backend-serverless.result }}"
          echo "- Admin Control Validation: ${{ needs.admin-control-result }}"
         
