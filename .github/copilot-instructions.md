# Choreo 3-Tier Application Deployment Guide

This rule provides comprehensive guidelines for AI agents to deploy 3-tier applications (frontend, backend, database) on the Choreo platform using MCP tools.

## When to Use This Rule

Apply this rule whenever:
- User mentions "Choreo" deployment
- User requests deployment of web applications with backend services
- User asks about 3-tier application deployment
- User mentions MCP tools for Choreo platform

## Deployment Workflow Overview

### Step 1: Environment Discovery and Setup

**Get Active Organization:**
- Tool: `get_active_org`
- Capture: `ORG_UUID`

**Project Management:**
- Tool: `get_projects` with `org_uuid`
- Check for existing "Reading List project" or similar
- If not exists, create using `create_project`:
  - `project_name`: Descriptive name (e.g., "Reading List project")
  - `description`: Brief description mentioning "vibe deployed by choreo mcp server"
  - `region`: "US" or "EU"
  - `org_uuid`: `<ORG_UUID>`
- Capture: `PROJECT_UUID`

**Environment Discovery:**
- Tool: `get_project_environments` with `project_uuid`
- Find "Development" environment
- Capture: `DEVELOPMENT_ENVIRONMENT_UUID`
- Note: Development environment should exist by default in Choreo

### Step 2: Backend Service Deployment

**Component Discovery/Creation:**
- Tool: `get_components` with `project_uuid`
- Check for existing backend service component
- If not exists, create using `create_service_component`:
  - `project_uuid`: `<PROJECT_UUID>`
  - `name`: "reading-list-service" (or appropriate service name)
  - `repository_url`: Current repository URL (get from context)
  - `repository_branch`: "main"
  - `repository_component_directory`: "backend"
  - `description`: Descriptive service description
  - `buildpack_id`: Get from `get_buildpacks(type="service")` - select Node.js
  - `buildpack_language_version`: "20.x.x" (or latest stable)
  - `endpoint_port`: `8080`
  - `endpoint_type`: "REST"
  - `endpoint_base_path`: "/"
- Capture: `BACKEND_COMPONENT_UUID`, `BACKEND_DEPLOYMENT_TRACK_ID`

**Build Management:**
- For new components: Monitor automatic build with `get_builds` (wait_for_completion=true)
- For existing components:
  - Get latest commit: `get_commit_history`
  - Check build status: `get_builds`
  - If no successful build for latest commit, trigger: `create_build`
- Capture: `BACKEND_BUILD_IMAGE_ID`

**Deployment:**
- Check current deployment: `get_deployment`
- If not active or outdated, deploy: `create_deployment`
- Capture: `BACKEND_SERVICE_URL` from deployment invokeUrl

### Step 3: Frontend Web Application Deployment

**Component Discovery/Creation:**
- Tool: `get_components` with `project_uuid`
- Check for existing web app component
- If not exists, create using `create_webapp_component`:
  - `project_uuid`: `<PROJECT_UUID>`
  - `name`: "reading-list-web-app" (or appropriate name)
  - `displayName`: User-friendly display name
  - `repoUrl`: Current repository URL
  - `branch`: "main"
  - `componentDir`: "frontend"
  - `buildPackLang`: "React" (get from `get_buildpacks(type="webApp")`)
  - `langVersion`: "20" (or appropriate Node.js version)
  - `port`: "8080"
  - `spaBuildCommand`: "npm install && npm run build"
  - `spaNodeVersion`: "20"
  - `spaOutputDir`: "dist"
- Capture: `FRONTEND_COMPONENT_UUID`, `FRONTEND_DEPLOYMENT_TRACK_ID`

**Configuration Setup:**
- Tool: `create_configurations`
- Create frontend API configuration:
  - `configuration_name`: "frontend-api-config"
  - `configuration_type`: "config-map"
  - `mount_type`: "file mount"
  - `file_mount_path`: "/app/public/config.js"
  - `file_mount_content`:
    ```javascript
    window.configs = {
        apiUrl: '<CONNECTION_URL>',
    };
    ```
  - IMPORANT: AGENT must prompt user for the connection URL it can NOT be derived from the backend service URL.

**Build and Deploy:**
- Follow same build/deploy pattern as backend
- Monitor builds, trigger if needed
- Deploy to Development environment
- Capture: `FRONTEND_APP_URL`

### Step 4: Post-Deployment Verification

**Test User Management:**
- Tool: `get_test_users` with `environment_uuid`
- If no test users exist, create using `add_test_user`:
  - `username`: "testuser"
  - `password`: "testpassword"
  - `email`: "test@example.com"
  - `groups`: "admin,internal"

**Final Verification:**
- Get final deployment URLs
- Present deployed application URL and test credentials to user

## Key MCP Tools Reference

| Tool | Purpose | Key Parameters |
|------|---------|----------------|
| `get_active_org` | Get organization UUID | None |
| `get_projects` | List/find projects | `org_uuid` |
| `create_project` | Create new project | `project_name`, `description`, `region`, `org_uuid` |
| `get_project_environments` | Get environments | `project_uuid` |
| `get_components` | List components | `project_uuid` |
| `create_service_component` | Create backend service | Multiple - see Step 2 |
| `create_webapp_component` | Create frontend app | Multiple - see Step 3 |
| `get_buildpacks` | Get available buildpacks | `type` ("service" or "webApp") |
| `get_builds` | Check build status | `component_uuid`, `project_uuid`, etc. |
| `create_build` | Trigger new build | `component_uuid`, `commit_hash`, etc. |
| `get_deployment` | Check deployment status | `component_uuid`, `project_uuid`, `environment_uuid` |
| `create_deployment` | Deploy component | `component_uuid`, `build_ref_or_image_id`, etc. |
| `create_configurations` | Set app configurations | `component_uuid`, configuration details |
| `get_test_users` | List test users | `environment_uuid` |
| `add_test_user` | Create test user | `username`, `password`, `email`, `groups` |

## Best Practices

1. **Always capture UUIDs** for reuse across deployment steps
2. **Check existing resources** before creating new ones
3. **Monitor build completion** before proceeding to deployment
4. **Verify deployment status** before considering step complete
5. **Use wait_for_completion=true** when monitoring builds
6. **Create meaningful names** that include project context
7. **Set appropriate buildpack versions** based on project requirements
8. **Configure frontend-backend connectivity** via config files

## Error Handling

- If "Development" environment doesn't exist, inform user it's required
- If builds fail, check buildpack compatibility and source code
- If deployments fail, verify build images and environment settings
- Always provide clear status updates during long-running operations

## Repository Structure Assumptions

- Backend code in `/backend` directory
- Frontend code in `/frontend` directory
- Backend runs on port 8080
- Frontend builds to `/dist` directory
- Frontend uses npm for package management

## Sequential Deployment Steps

1. **Environment Setup**: Get org UUID → Find/create project → Get development environment
2. **Backend Deployment**: Find/create service component → Build → Deploy → Get service URL
3. **Frontend Deployment**: Find/create webapp component → Configure API endpoint → Build → Deploy
4. **Verification**: Create test users → Get final URLs → Present to user

## Critical Success Factors

- Each step must complete successfully before proceeding
- Always verify deployment status is "ACTIVE"
- Ensure frontend can connect to backend via configuration
- Provide clear deployment URLs and test credentials at completion
