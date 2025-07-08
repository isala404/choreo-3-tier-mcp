### Choreo Deployment Guidelines for AI Agents

This section outlines the steps and the corresponding Choreo MCP tools for deploying this 3-tier application.

**Deployment Workflow:**

The agent should follow these steps sequentially:

#### Step 1: Discover Choreo Environment Details

Before creating components, the agent must identify the target Choreo organization, project, and environment.

*   **Action:** Get the active organization UUID.
    *   **MCP Tool:** `get_active_org`
    *   **Expected Output:** `ORG_UUID`
*   **Action:** Get available projects in the organization.
    *   **MCP Tool:** `get_projects` (with `org_uuid`)
    *   **Expected Output:** `PROJECT_UUID` for "Sample project" or the project it needs to create.
    *   **If project does not exist:**
        *   **Action:** Create the project.
        *   **MCP Tool:** `create_project`
        *   **Parameters:**
            *   `project_name`: "Sample project"
            *   `description`: "My sample project"
            *   `region`: "US" (or "EU" if preferred)
            *   `org_uuid`: `<ORG_UUID>`
*   **Action:** Get available environments for the project.
    *   **MCP Tool:** `get_project_environments`
    *   **Parameters:**
        *   `project_uuid`: `<PROJECT_UUID>`
    *   **Expected Output:** `ENVIRONMENT_UUID` for the "Development" environment.

#### Step 2: Deploy the Backend Service (`reading-list-service`)

*   **Action:** Create the Service Component in Choreo.
    *   **MCP Tool:** `create_service_component`
    *   **Parameters:**
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `name`: "reading-list-service"
        *   `repository_url`: "<CURRENT_REPO_URL>"
        *   `repository_branch`: "main"
        *   `repository_component_directory`: "backend"
        *   `description`: "Backend service for the reading list application"
        *   `buildpack_id`: (Retrieve using `get_buildpacks` for `service` type, e.g., Node.js buildpack ID)
        *   `buildpack_language_version`: "20.x.x" (or specific version ID)
        *   `endpoint_port`: `8080`
        *   `endpoint_type`: "REST"
        *   `endpoint_base_path`: "/"
*   **Action:** Get the component details to retrieve `component_uuid` and `component_deployment_track_id`.
    *   **MCP Tool:** `get_component` (or `get_components` and filter)
    *   **Parameters:**
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_uuid`: (Retrieve by name "reading-list-service" after creation)
*   **Action:** Trigger a build for the backend service.
    *   **MCP Tool:** `create_build`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<BACKEND_DEPLOYMENT_TRACK_ID>` (from `get_component` output)
        *   `commit_hash`: (Get latest commit hash using `get_commit_history` for the component's branch)
*   **Action:** Monitor the build status until `completed`.
    *   **MCP Tool:** `get_builds`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<BACKEND_DEPLOYMENT_TRACK_ID>`
    *   **Expected Output:** Look for `status: "completed"` and capture `image_id` from the latest successful build.
*   **Action:** Deploy the backend service to the Development environment.
    *   **MCP Tool:** `create_deployment`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<BACKEND_DEPLOYMENT_TRACK_ID>`
        *   `commit_hash`: (Same commit hash used for build)
        *   `build_ref_or_image_id`: `<BACKEND_BUILD_IMAGE_ID>` (from successful build)
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
*   **Action:** Get deployment details to retrieve the service `invokeUrl`.
    *   **MCP Tool:** `get_deployment`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
    *   **Expected Output:** Capture the `invokeUrl` for the backend service: `<BACKEND_SERVICE_URL>`.

#### Step 3: Deploy the Frontend Web Application (`reading-list-web-app`)

*   **Action:** Create the Web Application Component in Choreo.
    *   **MCP Tool:** `create_webapp_component`
    *   **Parameters:**
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `name`: "reading-list-web-app"
        *   `displayName`: "Reading List Web App"
        *   `repoUrl`: "<CURRENT_REPO_URL>"
        *   `branch`: "main"
        *   `componentDir`: "frontend"
        *   `buildPackLang`: "React" (or exact buildpack ID for React from `get_buildpacks`)
        *   `langVersion`: "18" (or get the latest version from `get_buildpacks`)
        *   `port`: "8080"
        *   `spaBuildCommand`: "npm install && npm run build"
        *   `spaNodeVersion`: "18"
        *   `spaOutputDir`: "dist"
*   **Action:** Get the component details to retrieve `component_uuid` and `component_deployment_track_id`.
    *   **MCP Tool:** `get_component` (or `get_components` and filter)
    *   **Parameters:**
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_uuid`: (Retrieve by name "reading-list-web-app" after creation)
*   **Action:** Configure the `config.js` file for the frontend to point to the backend service. This involves a file mount.
    *   **MCP Tool:** `create_configurations`
    *   **Parameters:**
        *   `component_uuid`: `<FRONTEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_id`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
        *   `component_deployment_track_id`: `<FRONTEND_DEPLOYMENT_TRACK_ID>`
        *   `configuration_name`: "frontend-api-config"
        *   `configuration_type`: "config-map"
        *   `mount_type`: "file mount"
        *   `file_mount_path`: "/app/public/config.js" (or `./public/config.js` depending on app's root in deployed environment)
        *   `file_mount_content`:
            ```javascript
            window.configs = {
                apiUrl: '<BACKEND_SERVICE_URL>', // Use the URL obtained from backend deployment
            };
            ```
            **Note:** The agent needs to substitute `<BACKEND_SERVICE_URL>` with the actual URL obtained from the backend's `get_deployment` tool call.
*   **Action:** Trigger a build for the frontend web application.
    *   **MCP Tool:** `create_build`
    *   **Parameters:**
        *   `component_uuid`: `<FRONTEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<FRONTEND_DEPLOYMENT_TRACK_ID>`
        *   `commit_hash`: (Get latest commit hash using `get_commit_history` for the component's branch)
*   **Action:** Monitor the build status until `completed`.
    *   **MCP Tool:** `get_builds`
    *   **Parameters:**
        *   `component_uuid`: `<FRONTEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<FRONTEND_DEPLOYMENT_TRACK_ID>`
    *   **Expected Output:** Look for `status: "completed"` and capture `image_id` from the latest successful build.
*   **Action:** Deploy the frontend web application to the Development environment.
    *   **MCP Tool:** `create_deployment`
    *   **Parameters:**
        *   `component_uuid`: `<FRONTEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `component_deployment_track_id`: `<FRONTEND_DEPLOYMENT_TRACK_ID>`
        *   `commit_hash`: (Same commit hash used for build)
        *   `build_ref_or_image_id`: `<FRONTEND_BUILD_IMAGE_ID>` (from successful build)
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`

#### Step 4: Post-Deployment Actions & Verification

*   **Action:** Get the deployed web application URL.
    *   **MCP Tool:** `get_deployment`
    *   **Parameters:**
        *   `component_uuid`: `<FRONTEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
    *   **Expected Output:** Capture `invokeUrl` (the Web App URL).
*   **Action:** View application logs for debugging.
    *   **MCP Tool:** `application_logs`
    *   **Parameters:**
        *   `component_uuid`: `<COMPONENT_UUID>` (either backend or frontend)
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
        *   `component_deployment_track_id`: `<COMPONENT_DEPLOYMENT_TRACK_ID>`
*   **Action:** View gateway logs (for backend service).
    *   **MCP Tool:** `gateway_logs`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
        *   `component_deployment_track_id`: `<BACKEND_DEPLOYMENT_TRACK_ID>`
*   **Action:** Generate a test key for the backend service (if needed for direct API testing).
    *   **MCP Tool:** `generate-test-key`
    *   **Parameters:**
        *   `component_uuid`: `<BACKEND_COMPONENT_UUID>`
        *   `project_uuid`: `<PROJECT_UUID>`
        *   `environment_uuid`: `<DEVELOPMENT_ENVIRONMENT_UUID>`
