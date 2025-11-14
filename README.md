# Jenkins MCP Server

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Python](https://img.shields.io/badge/Python-3.12+-blue.svg)](https://www.python.org/downloads/)
[![Node.js](https://img.shields.io/badge/Node.js-14.0+-green.svg)](https://nodejs.org/)

An enterprise-grade MCP (Model Context Protocol) server for Jenkins CI/CD integration with advanced features including multi-tier caching, pipeline monitoring, artifact management, and batch operations.

## Features

### Core Capabilities
- **Job Management**: Trigger, list, search, and monitor Jenkins jobs with folder support
- **Build Status**: Real-time build status tracking and console log streaming
- **Pipeline Support**: Stage-by-stage pipeline execution monitoring with detailed logs
- **Artifact Management**: List, download, and search build artifacts across builds
- **Batch Operations**: Parallel job execution with priority queuing
- **Advanced Filtering**: Filter jobs by status, results, dates, and more with regex support

### Performance & Reliability
- **Multi-Tier Caching System**: 5-tier intelligent caching (static, semi-static, dynamic, short-lived, permanent)
- **Retry Logic**: Built-in exponential backoff for improved reliability
- **CSRF Protection**: Automatic crumb token handling
- **Input Validation**: Robust Pydantic-based validation

### Enterprise Features
- **Nested Job Support**: Full support for Jenkins folder structures
- **2FA Compatible**: Works with Jenkins two-factor authentication
- **Queue Management**: Real-time build queue monitoring
- **Transport Flexibility**: Supports STDIO and Streamable HTTP transports

## Prerequisites

- **Node.js**: 14.0.0 or higher
- **Python**: 3.12 or higher
- **Jenkins**: 2.401+ (recommended)
- **Jenkins API Token**: Required for authentication

## Installation

### Clone and Install

```bash
# Clone this repository
git clone https://github.com/avisangle/jenkins-mcp-server
cd jenkins-mcp-server

# Install Node.js dependencies
npm install

# Install Python dependencies
pip install -e .
# Or using uv (recommended)
uv pip install -e .

# Verify installation
node bin/jenkins-mcp.js --help
```

### Dependencies

**Python packages** (automatically installed):
- `mcp[cli]>=1.11.0` - Model Context Protocol
- `requests>=2.32.4` - HTTP library
- `cachetools>=5.5.0` - Caching utilities
- `python-dotenv` - Environment variable management
- `pydantic` - Data validation
- `fastapi` - HTTP transport support

## Configuration

### Environment Variables

Create a `.env` file in your working directory:

```bash
# Required Jenkins Configuration
JENKINS_URL="http://your-jenkins-server:8080"
JENKINS_USER="your-username"
JENKINS_API_TOKEN="your-api-token"

# Optional: Retry Configuration
JENKINS_MAX_RETRIES=3
JENKINS_RETRY_BASE_DELAY=1.0
JENKINS_RETRY_MAX_DELAY=60.0
JENKINS_RETRY_BACKOFF_MULTIPLIER=2.0

# Optional: Cache Configuration
JENKINS_CACHE_STATIC_TTL=3600        # 1 hour for static data
JENKINS_CACHE_SEMI_STATIC_TTL=300    # 5 minutes for semi-static
JENKINS_CACHE_DYNAMIC_TTL=30         # 30 seconds for dynamic data
JENKINS_CACHE_SHORT_TTL=10           # 10 seconds for short-lived

# Optional: Cache Size Limits
JENKINS_CACHE_STATIC_SIZE=1000
JENKINS_CACHE_SEMI_STATIC_SIZE=500
JENKINS_CACHE_DYNAMIC_SIZE=200
JENKINS_CACHE_PERMANENT_SIZE=2000
JENKINS_CACHE_SHORT_SIZE=100

# Optional: Content Limits
JENKINS_MAX_LOG_SIZE=1000
JENKINS_MAX_CONTENT_SIZE=10000
JENKINS_MAX_ARTIFACT_SIZE_MB=50

# Optional: Server Configuration
MCP_PORT=8010
MCP_HOST=0.0.0.0
```

### Getting a Jenkins API Token

1. Log into your Jenkins server
2. Click on your username → **Configure**
3. Under **API Token** section, click **Add new Token**
4. Give it a name and click **Generate**
5. Copy the generated token (you won't be able to see it again)

## Usage

### Claude Desktop Integration

Add to your `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "jenkins": {
      "command": "node",
      "args": [
        "/path/to/jenkins-mcp-server/bin/jenkins-mcp.js"
      ],
      "env": {
        "JENKINS_URL": "http://your-jenkins-server:8080",
        "JENKINS_USER": "your-username",
        "JENKINS_API_TOKEN": "your-api-token"
      }
    }
  }
}
```

### Command Line

```bash
# STDIO mode (for Claude Desktop/MCP clients)
node bin/jenkins-mcp.js

# HTTP mode (for MCP Gateway)
node bin/jenkins-mcp.js --transport streamable-http --port 8010

# With custom host
node bin/jenkins-mcp.js --transport streamable-http --port 8080 --host localhost
```

### Transport Modes

**STDIO Mode** (default):
- Standard input/output communication
- Best for Claude Desktop integration
- No network ports required

**Streamable HTTP Mode**:
- HTTP-based communication
- Useful for MCP Gateway or remote connections
- Requires port configuration

## Available Tools

The server exposes 21 MCP tools for Jenkins interaction:

### Job Management

#### `trigger_job`
Trigger a Jenkins job with optional parameters.

**Parameters:**
- `job_name` (string): Job name or path (e.g., `"my-job"` or `"folder/my-job"`)
- `params` (object, optional): Job parameters as key-value pairs

**Example:**
```json
{
  "job_name": "build-project",
  "params": {
    "BRANCH": "main",
    "VERSION": "1.0.0"
  }
}
```

#### `get_job_info`
Get detailed information about a Jenkins job.

**Parameters:**
- `job_name` (string): Job name or path
- `auto_search` (boolean, default: true): Perform pattern search if direct lookup fails

**Returns:** Job configuration, parameters, last build info, and health status

#### `list_jobs`
List Jenkins jobs with advanced filtering.

**Parameters:**
- `recursive` (boolean, default: true): Include jobs in subfolders
- `max_depth` (integer, default: 10): Maximum folder depth
- `include_folders` (boolean, default: false): Include folder items
- `status_filter` (string, optional): Filter by color (e.g., `"blue"`, `"red"`)
- `last_build_result` (string, optional): Filter by result (`"SUCCESS"`, `"FAILURE"`, `"UNSTABLE"`)
- `days_since_last_build` (integer, optional): Filter by days since last build
- `enabled_only` (boolean, optional): Show only enabled jobs

**Returns:** Hierarchical list of jobs with statistics

#### `search_jobs`
Search for Jenkins jobs by pattern.

**Parameters:**
- `pattern` (string): Search pattern (supports wildcards)
- `job_type` (string, default: `"job"`): Type filter (`"job"`, `"folder"`, `"pipeline"`)
- `use_regex` (boolean, default: false): Use regex pattern matching
- `max_depth` (integer, default: 10): Maximum search depth

**Returns:** List of matching jobs with details

#### `search_and_trigger`
Search for jobs and trigger all matches.

**Parameters:**
- `pattern` (string): Search pattern
- `params` (object, optional): Parameters for all jobs
- `max_depth` (integer, default: 10): Maximum search depth

**Returns:** Trigger status for each matched job

#### `get_folder_info`
Get information about a Jenkins folder.

**Parameters:**
- `folder_path` (string): Folder path (e.g., `"team/project"`)

**Returns:** Folder details and contained jobs

### Build Status & Monitoring

#### `get_build_status`
Get status of a specific build.

**Parameters:**
- `job_name` (string): Job name or path
- `build_number` (integer): Build number

**Returns:** Build status, result, duration, timestamp, and executor info

#### `get_console_log`
Get console output from a build.

**Parameters:**
- `job_name` (string): Job name or path
- `build_number` (integer): Build number
- `start` (integer, default: 0): Starting byte offset

**Returns:** Console log content and metadata

#### `summarize_build_log`
Get a summarized version of build console log.

**Parameters:**
- `job_name` (string): Job name or path
- `build_number` (integer): Build number

**Returns:** Summarized log content

### Pipeline Support

#### `get_pipeline_status`
Get detailed pipeline execution status with stage information.

**Parameters:**
- `job_name` (string): Pipeline job name
- `build_number` (integer): Build number

**Returns:** Pipeline stages with status, duration, and logs for each stage

### Artifact Management

#### `list_build_artifacts`
List all artifacts from a build.

**Parameters:**
- `job_name` (string): Job name or path
- `build_number` (integer): Build number

**Returns:** List of artifacts with paths, sizes, and download URLs

#### `download_build_artifact`
Download a specific build artifact.

**Parameters:**
- `job_name` (string): Job name or path
- `build_number` (integer): Build number
- `artifact_path` (string): Relative path to artifact
- `max_size_mb` (integer, default: 50): Maximum download size in MB

**Returns:** Artifact content (base64 encoded for binary) or text content

#### `search_build_artifacts`
Search for artifacts across multiple builds.

**Parameters:**
- `job_name` (string): Job name or path
- `pattern` (string): Search pattern for artifact names
- `max_builds` (integer, default: 10): Maximum builds to search
- `use_regex` (boolean, default: false): Use regex matching

**Returns:** List of matching artifacts across builds

### Batch Operations

#### `batch_trigger_jobs`
Trigger multiple jobs in parallel with priority queuing.

**Parameters:**
- `operations` (array): List of job operations
  - `job_name` (string): Job name
  - `params` (object, optional): Job parameters
  - `priority` (integer, default: 0): Execution priority (higher = earlier)
- `max_concurrent` (integer, default: 5): Maximum parallel executions
- `operation_name` (string, optional): Name for this batch operation

**Returns:** Operation ID for monitoring

**Example:**
```json
{
  "operations": [
    {"job_name": "build-frontend", "priority": 10},
    {"job_name": "build-backend", "priority": 10},
    {"job_name": "integration-tests", "priority": 5}
  ],
  "max_concurrent": 2
}
```

#### `batch_monitor_jobs`
Monitor the status of a batch operation.

**Parameters:**
- `operation_id` (string): Operation ID from batch_trigger_jobs

**Returns:** Status of each job in the batch operation

#### `batch_cancel_jobs`
Cancel a batch operation.

**Parameters:**
- `operation_id` (string): Operation ID to cancel
- `cancel_running_builds` (boolean, default: false): Also cancel running builds

**Returns:** Cancellation status

### Queue & Server Management

#### `get_queue_info`
Get information about the Jenkins build queue.

**Returns:** List of queued items with job names, wait times, and reasons

#### `server_info`
Get basic Jenkins server information.

**Returns:** Version, root URL, and system information

### Cache Management

#### `get_cache_statistics`
Get statistics about the caching system.

**Returns:** Cache hit/miss rates, sizes, and memory usage

#### `clear_cache`
Clear cached data.

**Parameters:**
- `cache_type` (string, optional): Specific cache to clear (`"static"`, `"dynamic"`, etc.)
- `job_name` (string, optional): Clear cache for specific job

**Returns:** Cache clear status

#### `warm_cache`
Pre-populate cache with commonly used data.

**Parameters:**
- `operations` (array, optional): List of operations to warm (`["list_jobs"`, `"server_info"]`)

**Returns:** Warm-up status and timing

## Caching System

The server implements a sophisticated 5-tier caching system for optimal performance:

### Cache Tiers

| Tier | TTL | Use Case | Example Data |
|------|-----|----------|--------------|
| **Static** | 1 hour | Rarely changing data | Job definitions, server info |
| **Semi-Static** | 5 minutes | Occasionally changing | Job lists, folder structures |
| **Dynamic** | 30 seconds | Frequently changing | Build statuses, queue info |
| **Short-Lived** | 10 seconds | Real-time data | Active build logs |
| **Permanent** | No expiry | Immutable data | Completed build artifacts |

### Cache Invalidation

Caches are automatically invalidated when:
- Jobs are triggered
- Build states change
- Manual cache clear operations

### Cache Configuration

Customize cache behavior via environment variables (see Configuration section).

## Error Handling

The server provides detailed error messages with suggestions:

- **404 Not Found**: Includes search suggestions for similar job names
- **401 Unauthorized**: Prompts to check credentials
- **403 Forbidden**: Explains permission issues
- **Connection Errors**: Automatic retry with exponential backoff

## Nested Job Support

Fully supports Jenkins folder structures:

```json
{
  "job_name": "team-alpha/backend/build-service"
}
```

All tools accept nested paths with automatic path handling.

## Performance Optimization

### Built-in Optimizations
- Multi-tier intelligent caching
- Concurrent batch operations with throttling
- Request deduplication
- Lazy loading of large datasets
- Configurable content size limits

### Best Practices
1. Use batch operations for multiple jobs
2. Enable caching for repeated operations
3. Set appropriate `max_depth` when searching
4. Use `status_filter` to reduce data transfer
5. Warm cache before high-traffic periods

## Security

### Authentication
- Jenkins API token authentication
- Support for Jenkins 2FA environments
- Automatic CSRF crumb token handling

### Security Features
- No password storage (API tokens only)
- TLS/SSL support for Jenkins HTTPS endpoints
- Input validation on all parameters
- Configurable artifact download size limits

## Troubleshooting

### Common Issues

**Python version error:**
```bash
# Check Python version
python3 --version  # Must be 3.12+
```

**Missing dependencies:**
```bash
# Reinstall Python packages
pip install -e .
# Or
uv pip install -e .
```

**Connection refused:**
- Verify `JENKINS_URL` is correct and accessible
- Check Jenkins server is running
- Verify network connectivity

**Authentication failed:**
- Regenerate Jenkins API token
- Check `JENKINS_USER` matches your Jenkins username
- Verify token has necessary permissions

**Job not found:**
- Check job name spelling and case
- Use `search_jobs` to find correct job name
- Verify you have permissions to access the job

### Debug Logging

The server logs to stderr with request IDs for troubleshooting:

```
2025-11-12 10:30:45 - jenkins_mcp - INFO - [req-abc123] Received request to trigger job: 'build-project'
```

## Development

### Running Tests

```bash
# Install test dependencies
pip install -e ".[test]"

# Run tests
pytest
```

### Project Structure

```
jenkins-mcp-server/
├── bin/
│   └── jenkins-mcp.js          # Node.js CLI entry point
├── python/
│   └── jenkins_mcp_server_enhanced.py  # Main MCP server implementation
├── scripts/
│   └── check-python.js         # Python environment checker
├── .env.example                # Example environment configuration
├── pyproject.toml              # Python package configuration
├── package.json                # Node.js package configuration
└── README.md                   # This file
```

## Contributing

Contributions, bug reports, and feature requests are welcome.

## License

Apache License 2.0 - See [LICENSE](LICENSE) file for details.

## Related Projects

- **MCP Protocol**: [Model Context Protocol](https://modelcontextprotocol.io/)
- **Claude Desktop**: [Anthropic Claude](https://www.anthropic.com/claude)

## Support

For issues and questions:
- Open an issue on [GitHub Issues](https://github.com/avisangle/jenkins-mcp-server/issues)
- Check existing issues for solutions
- Provide Jenkins version, MCP server version, and error logs when reporting issues

---

**Version**: 1.0.7
**Last Updated**: 2025-11-13
