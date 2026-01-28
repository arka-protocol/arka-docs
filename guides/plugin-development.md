# Plugin Development Guide

This guide covers how to develop custom plugins for the ARKA Engine to extend compliance evaluation capabilities.

## Overview

Plugins allow you to extend the ARKA Engine with:
- Custom data sources and enrichment
- Domain-specific compliance logic
- External system integrations
- Custom actions and notifications

```
┌─────────────────────────────────────────────────────────────┐
│                    Plugin Architecture                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────────────────────────────────────────────┐│
│  │                    ARKA Engine                           ││
│  │  ┌─────────────────────────────────────────────────────┐││
│  │  │                     RTVM                             │││
│  │  │    ┌─────────┐  ┌─────────┐  ┌─────────────────┐   │││
│  │  │    │  Rule   │  │  Plugin │  │  Plugin         │   │││
│  │  │    │ Executor│◀─│ Registry│──│  Sandbox        │   │││
│  │  │    └─────────┘  └─────────┘  └─────────────────┘   │││
│  │  └─────────────────────────────────────────────────────┘││
│  └─────────────────────────────────────────────────────────┘│
│                            │                                 │
│         ┌──────────────────┼──────────────────┐             │
│         │                  │                  │             │
│         ▼                  ▼                  ▼             │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐       │
│  │  AML Plugin │   │ KYC Plugin  │   │Custom Plugin│       │
│  └─────────────┘   └─────────────┘   └─────────────┘       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Plugin Types

### Data Source Plugins

Fetch and enrich data from external sources:

```python
from pact.plugins import DataSourcePlugin, DataSourceResult
from pact.plugins.decorators import plugin, data_source

@plugin(
    name="sanctions-list",
    version="1.0.0",
    description="OFAC Sanctions List Data Source"
)
class SanctionsListPlugin(DataSourcePlugin):
    """Plugin to check entities against sanctions lists."""

    def __init__(self, config: dict):
        self.api_endpoint = config.get("api_endpoint")
        self.api_key = config.get("api_key")
        self.cache_ttl = config.get("cache_ttl", 3600)

    @data_source(
        name="check_sanctions",
        description="Check entity against sanctions lists",
        cache_key=lambda entity_id: f"sanctions:{entity_id}",
        cache_ttl=3600
    )
    async def check_sanctions(
        self,
        entity_id: str,
        entity_type: str,
        lists: list[str] = None
    ) -> DataSourceResult:
        """
        Check an entity against sanctions lists.

        Args:
            entity_id: The entity identifier
            entity_type: Type of entity (individual, organization)
            lists: Specific lists to check (default: all)

        Returns:
            DataSourceResult with sanctions match information
        """
        # Call external API
        response = await self._call_api(
            endpoint="/sanctions/check",
            params={
                "entity_id": entity_id,
                "entity_type": entity_type,
                "lists": lists or ["OFAC", "UN", "EU"]
            }
        )

        return DataSourceResult(
            success=True,
            data={
                "is_sanctioned": response["match"],
                "matches": response.get("matches", []),
                "lists_checked": response["lists"],
                "check_timestamp": response["timestamp"]
            },
            metadata={
                "source": "sanctions-api",
                "version": response["api_version"]
            }
        )

    async def _call_api(self, endpoint: str, params: dict) -> dict:
        async with aiohttp.ClientSession() as session:
            async with session.get(
                f"{self.api_endpoint}{endpoint}",
                params=params,
                headers={"Authorization": f"Bearer {self.api_key}"}
            ) as response:
                return await response.json()
```

### Enrichment Plugins

Transform and enrich transaction data:

```python
from pact.plugins import EnrichmentPlugin, EnrichmentResult
from pact.plugins.decorators import plugin, enrichment

@plugin(
    name="geo-enrichment",
    version="1.0.0",
    description="Geographic data enrichment"
)
class GeoEnrichmentPlugin(EnrichmentPlugin):
    """Enrich transactions with geographic risk data."""

    def __init__(self, config: dict):
        self.risk_data = self._load_risk_data(config["risk_data_path"])

    @enrichment(
        name="enrich_geography",
        description="Add geographic risk scores",
        input_schema={
            "type": "object",
            "properties": {
                "country_code": {"type": "string"}
            },
            "required": ["country_code"]
        }
    )
    async def enrich_geography(
        self,
        country_code: str,
        context: dict
    ) -> EnrichmentResult:
        """
        Enrich with geographic risk information.

        Args:
            country_code: ISO country code
            context: Evaluation context

        Returns:
            EnrichmentResult with geographic risk data
        """
        risk_data = self.risk_data.get(country_code, {})

        return EnrichmentResult(
            success=True,
            enrichments={
                "geo_risk_score": risk_data.get("risk_score", 50),
                "geo_risk_category": risk_data.get("category", "medium"),
                "is_fatf_high_risk": risk_data.get("fatf_high_risk", False),
                "is_tax_haven": risk_data.get("tax_haven", False),
                "aml_regime_strength": risk_data.get("aml_strength", "moderate")
            }
        )

    def _load_risk_data(self, path: str) -> dict:
        with open(path) as f:
            return json.load(f)
```

### Action Plugins

Execute custom actions based on rule outcomes:

```python
from pact.plugins import ActionPlugin, ActionResult
from pact.plugins.decorators import plugin, action

@plugin(
    name="alert-system",
    version="1.0.0",
    description="Multi-channel alerting system"
)
class AlertSystemPlugin(ActionPlugin):
    """Send alerts to various channels."""

    def __init__(self, config: dict):
        self.slack_webhook = config.get("slack_webhook")
        self.email_config = config.get("email")
        self.pagerduty_key = config.get("pagerduty_key")

    @action(
        name="send_alert",
        description="Send alert to configured channels",
        async_execution=True
    )
    async def send_alert(
        self,
        severity: str,
        message: str,
        channels: list[str],
        context: dict
    ) -> ActionResult:
        """
        Send alert to specified channels.

        Args:
            severity: Alert severity (low, medium, high, critical)
            message: Alert message
            channels: List of channels (slack, email, pagerduty)
            context: Evaluation context

        Returns:
            ActionResult indicating success/failure
        """
        results = {}

        if "slack" in channels:
            results["slack"] = await self._send_slack(severity, message, context)

        if "email" in channels:
            results["email"] = await self._send_email(severity, message, context)

        if "pagerduty" in channels and severity in ["high", "critical"]:
            results["pagerduty"] = await self._send_pagerduty(severity, message, context)

        all_success = all(r["success"] for r in results.values())

        return ActionResult(
            success=all_success,
            data={"channel_results": results},
            error=None if all_success else "Some channels failed"
        )

    async def _send_slack(self, severity, message, context) -> dict:
        color = {"low": "#36a64f", "medium": "#f2c744", "high": "#ff9900", "critical": "#ff0000"}

        payload = {
            "attachments": [{
                "color": color.get(severity, "#808080"),
                "title": f"Compliance Alert: {severity.upper()}",
                "text": message,
                "fields": [
                    {"title": "Transaction ID", "value": context.get("transaction_id"), "short": True},
                    {"title": "Rule ID", "value": context.get("rule_id"), "short": True}
                ]
            }]
        }

        async with aiohttp.ClientSession() as session:
            async with session.post(self.slack_webhook, json=payload) as response:
                return {"success": response.status == 200}

    async def _send_email(self, severity, message, context) -> dict:
        # Email implementation
        pass

    async def _send_pagerduty(self, severity, message, context) -> dict:
        # PagerDuty implementation
        pass
```

### Validation Plugins

Custom validation logic:

```python
from pact.plugins import ValidationPlugin, ValidationResult
from pact.plugins.decorators import plugin, validator

@plugin(
    name="kyc-validation",
    version="1.0.0",
    description="KYC document validation"
)
class KYCValidationPlugin(ValidationPlugin):
    """Validate KYC documents and data."""

    def __init__(self, config: dict):
        self.document_api = config.get("document_api")
        self.ml_model = self._load_ml_model(config.get("model_path"))

    @validator(
        name="validate_identity_document",
        description="Validate identity document authenticity"
    )
    async def validate_identity_document(
        self,
        document_type: str,
        document_data: dict,
        context: dict
    ) -> ValidationResult:
        """
        Validate identity document.

        Args:
            document_type: Type of document (passport, drivers_license, etc.)
            document_data: Document data including image
            context: Evaluation context

        Returns:
            ValidationResult with validation details
        """
        # OCR extraction
        extracted = await self._extract_document_data(document_data["image"])

        # ML fraud detection
        fraud_score = self.ml_model.predict(document_data["image"])

        # Data consistency check
        consistency = self._check_consistency(extracted, document_data.get("claimed_data"))

        is_valid = (
            fraud_score < 0.3 and
            consistency["match_score"] > 0.9 and
            extracted.get("expiry_valid", False)
        )

        return ValidationResult(
            valid=is_valid,
            score=1.0 - fraud_score,
            details={
                "fraud_score": fraud_score,
                "consistency_score": consistency["match_score"],
                "extracted_data": extracted,
                "expiry_valid": extracted.get("expiry_valid"),
                "document_type_verified": extracted.get("type") == document_type
            },
            errors=[] if is_valid else ["Document validation failed"]
        )
```

## Plugin SDK

### Plugin Base Classes

```python
from abc import ABC, abstractmethod
from typing import Any, Dict, Optional
from dataclasses import dataclass

@dataclass
class PluginMetadata:
    """Plugin metadata."""
    name: str
    version: str
    description: str
    author: str = ""
    license: str = ""
    dependencies: list[str] = None
    config_schema: dict = None

class BasePlugin(ABC):
    """Base class for all plugins."""

    def __init__(self, config: dict):
        self.config = config
        self._initialized = False

    @property
    @abstractmethod
    def metadata(self) -> PluginMetadata:
        """Return plugin metadata."""
        pass

    async def initialize(self) -> None:
        """Initialize the plugin. Called once on load."""
        self._initialized = True

    async def shutdown(self) -> None:
        """Cleanup when plugin is unloaded."""
        pass

    async def health_check(self) -> dict:
        """Return plugin health status."""
        return {"healthy": self._initialized}

class DataSourcePlugin(BasePlugin):
    """Base class for data source plugins."""

    @abstractmethod
    async def fetch(self, **kwargs) -> DataSourceResult:
        """Fetch data from the source."""
        pass

class EnrichmentPlugin(BasePlugin):
    """Base class for enrichment plugins."""

    @abstractmethod
    async def enrich(self, data: dict, context: dict) -> EnrichmentResult:
        """Enrich input data."""
        pass

class ActionPlugin(BasePlugin):
    """Base class for action plugins."""

    @abstractmethod
    async def execute(self, action: str, params: dict, context: dict) -> ActionResult:
        """Execute an action."""
        pass

class ValidationPlugin(BasePlugin):
    """Base class for validation plugins."""

    @abstractmethod
    async def validate(self, data: dict, context: dict) -> ValidationResult:
        """Validate input data."""
        pass
```

### Result Types

```python
from dataclasses import dataclass
from typing import Any, Dict, List, Optional

@dataclass
class DataSourceResult:
    """Result from a data source query."""
    success: bool
    data: Dict[str, Any]
    metadata: Dict[str, Any] = None
    error: Optional[str] = None
    cached: bool = False
    cache_expires: Optional[datetime] = None

@dataclass
class EnrichmentResult:
    """Result from data enrichment."""
    success: bool
    enrichments: Dict[str, Any]
    warnings: List[str] = None
    error: Optional[str] = None

@dataclass
class ActionResult:
    """Result from action execution."""
    success: bool
    data: Dict[str, Any] = None
    error: Optional[str] = None
    async_task_id: Optional[str] = None

@dataclass
class ValidationResult:
    """Result from validation."""
    valid: bool
    score: float = 1.0
    details: Dict[str, Any] = None
    errors: List[str] = None
    warnings: List[str] = None
```

### Decorators

```python
from functools import wraps
from typing import Callable

def plugin(name: str, version: str, description: str, **kwargs):
    """Decorator to register a plugin class."""
    def decorator(cls):
        cls._plugin_metadata = PluginMetadata(
            name=name,
            version=version,
            description=description,
            **kwargs
        )

        @property
        def metadata(self) -> PluginMetadata:
            return cls._plugin_metadata

        cls.metadata = metadata
        return cls
    return decorator

def data_source(
    name: str,
    description: str,
    cache_key: Callable = None,
    cache_ttl: int = 0,
    timeout: int = 30
):
    """Decorator to register a data source method."""
    def decorator(func):
        func._data_source_metadata = {
            "name": name,
            "description": description,
            "cache_key": cache_key,
            "cache_ttl": cache_ttl,
            "timeout": timeout
        }

        @wraps(func)
        async def wrapper(self, *args, **kwargs):
            # Caching logic
            if cache_key and cache_ttl > 0:
                key = cache_key(*args, **kwargs)
                cached = await self._cache.get(key)
                if cached:
                    return DataSourceResult(
                        success=True,
                        data=cached,
                        cached=True
                    )

            # Execute function
            result = await func(self, *args, **kwargs)

            # Store in cache
            if cache_key and cache_ttl > 0 and result.success:
                await self._cache.set(key, result.data, ttl=cache_ttl)

            return result

        return wrapper
    return decorator

def action(
    name: str,
    description: str,
    async_execution: bool = False,
    retry_on_failure: bool = True,
    max_retries: int = 3
):
    """Decorator to register an action method."""
    def decorator(func):
        func._action_metadata = {
            "name": name,
            "description": description,
            "async_execution": async_execution,
            "retry_on_failure": retry_on_failure,
            "max_retries": max_retries
        }

        @wraps(func)
        async def wrapper(self, *args, **kwargs):
            if async_execution:
                # Queue for async execution
                task_id = await self._queue_async_action(func, args, kwargs)
                return ActionResult(
                    success=True,
                    async_task_id=task_id
                )

            # Synchronous execution with retry
            last_error = None
            for attempt in range(max_retries if retry_on_failure else 1):
                try:
                    return await func(self, *args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_retries - 1:
                        await asyncio.sleep(2 ** attempt)

            return ActionResult(
                success=False,
                error=str(last_error)
            )

        return wrapper
    return decorator
```

## Plugin Configuration

### Configuration Schema

```yaml
# plugin-config.yaml
apiVersion: arka.io/v1
kind: PluginConfig
metadata:
  name: sanctions-list
  namespace: plugins
spec:
  plugin: pact/sanctions-list
  version: "1.0.0"

  # Plugin-specific configuration
  config:
    api_endpoint: "https://api.sanctions-provider.com"
    api_key: ${SANCTIONS_API_KEY}
    cache_ttl: 3600
    lists:
      - OFAC
      - UN
      - EU

  # Resource limits
  resources:
    cpu: "100m"
    memory: "256Mi"
    timeout: 30s

  # Sandbox settings
  sandbox:
    networkAccess:
      allowed:
        - "api.sanctions-provider.com"
    fileAccess:
      read:
        - "/data/sanctions-cache"
      write: []

  # Logging
  logging:
    level: info
    includeRequestData: false
```

### Environment Variables

```python
from pact.plugins import PluginConfig

class SanctionsPlugin(DataSourcePlugin):
    def __init__(self, config: dict):
        # Config values can reference environment variables
        self.api_key = config.get("api_key")  # ${SANCTIONS_API_KEY}

        # Or use the config helper
        self.api_key = PluginConfig.get_env("SANCTIONS_API_KEY")

        # With fallback
        self.timeout = PluginConfig.get_env("SANCTIONS_TIMEOUT", default=30)
```

### Secrets Management

```python
from pact.plugins import SecretsManager

class SecurePlugin(BasePlugin):
    async def initialize(self):
        secrets = SecretsManager()

        # Fetch from configured secrets backend (Vault, AWS Secrets Manager, etc.)
        self.api_key = await secrets.get("sanctions/api-key")
        self.db_password = await secrets.get("database/password")
```

## Using Plugins in Rules

### DSL Integration

```
// Rule using sanctions check plugin
rule sanctions_screening {
    when {
        // Use plugin data source
        let sanctions_result = plugin.sanctions_list.check_sanctions(
            entity_id: transaction.sender.id,
            entity_type: "individual"
        )

        sanctions_result.is_sanctioned == true
    }
    then {
        // Use plugin action
        plugin.alert_system.send_alert(
            severity: "critical",
            message: "Sanctioned entity detected",
            channels: ["slack", "pagerduty"]
        )

        reject(reason: "Entity on sanctions list")
    }
}

// Rule using enrichment plugin
rule high_risk_geography {
    when {
        // Enrich with geographic risk data
        let geo_data = plugin.geo_enrichment.enrich_geography(
            country_code: transaction.destination.country
        )

        geo_data.geo_risk_score > 80 and
        transaction.amount > 10000
    }
    then {
        flag_for_review(reason: "High risk geography")
    }
}

// Rule using validation plugin
rule kyc_document_check {
    when {
        // Validate identity document
        let validation = plugin.kyc_validation.validate_identity_document(
            document_type: customer.document_type,
            document_data: customer.document
        )

        validation.valid == false or
        validation.score < 0.8
    }
    then {
        require_manual_review(
            reason: "Document validation failed",
            details: validation.details
        )
    }
}
```

### Programmatic Usage

```python
from pact import ARKAEngine
from pact.plugins import PluginManager

# Load plugins
plugin_manager = PluginManager()
plugin_manager.load("pact/sanctions-list", config={...})
plugin_manager.load("pact/geo-enrichment", config={...})

# Create engine with plugins
engine = ARKAEngine(
    plugin_manager=plugin_manager
)

# Plugins are available in evaluation context
result = engine.evaluate(
    transaction=transaction,
    plugins={
        "sanctions_list": True,
        "geo_enrichment": True
    }
)
```

## Plugin Development Workflow

### Project Structure

```
my-plugin/
├── src/
│   └── my_plugin/
│       ├── __init__.py
│       ├── plugin.py          # Main plugin class
│       ├── handlers/          # Request handlers
│       │   ├── __init__.py
│       │   └── sanctions.py
│       ├── models/            # Data models
│       │   ├── __init__.py
│       │   └── types.py
│       └── utils/             # Utilities
│           ├── __init__.py
│           └── helpers.py
├── tests/
│   ├── __init__.py
│   ├── test_plugin.py
│   └── fixtures/
│       └── sample_data.json
├── config/
│   └── default.yaml          # Default configuration
├── pyproject.toml
├── README.md
└── CHANGELOG.md
```

### Plugin Manifest

```yaml
# plugin.yaml
apiVersion: arka.io/v1
kind: PluginManifest
metadata:
  name: my-custom-plugin
  version: 1.0.0
  description: Custom compliance plugin
  author: Your Organization
  license: Apache-2.0

spec:
  # Entry point
  entrypoint: my_plugin.plugin:MyPlugin

  # Supported plugin types
  types:
    - data_source
    - action

  # Dependencies
  dependencies:
    python: ">=3.10"
    arka-sdk: ">=1.0.0"
    packages:
      - aiohttp>=3.8.0
      - pydantic>=2.0.0

  # Exposed functions
  exports:
    data_sources:
      - name: check_entity
        description: Check entity against custom database
        parameters:
          - name: entity_id
            type: string
            required: true
          - name: check_type
            type: string
            required: false
            default: "full"

    actions:
      - name: log_event
        description: Log compliance event
        parameters:
          - name: event_type
            type: string
            required: true
          - name: details
            type: object
            required: true

  # Required permissions
  permissions:
    network:
      - api.myservice.com
    secrets:
      - my-plugin/api-key
```

### Testing Plugins

```python
import pytest
from pact.plugins.testing import PluginTestHarness, MockContext

from my_plugin import MyPlugin

@pytest.fixture
def plugin():
    """Create plugin instance for testing."""
    config = {
        "api_endpoint": "https://mock-api.test",
        "api_key": "test-key"
    }
    plugin = MyPlugin(config)
    return plugin

@pytest.fixture
def harness(plugin):
    """Create test harness."""
    return PluginTestHarness(plugin)

class TestMyPlugin:
    async def test_data_source_success(self, harness):
        """Test successful data source call."""
        # Mock external API
        harness.mock_http_response(
            url="https://mock-api.test/check",
            response={"match": False, "score": 0.1}
        )

        # Call plugin function
        result = await harness.call_data_source(
            "check_entity",
            entity_id="entity-123"
        )

        assert result.success
        assert result.data["match"] == False

    async def test_data_source_error(self, harness):
        """Test error handling."""
        harness.mock_http_error(
            url="https://mock-api.test/check",
            status=500
        )

        result = await harness.call_data_source(
            "check_entity",
            entity_id="entity-123"
        )

        assert not result.success
        assert "error" in result.error.lower()

    async def test_action_execution(self, harness):
        """Test action execution."""
        harness.mock_http_response(
            url="https://mock-api.test/log",
            response={"logged": True}
        )

        result = await harness.call_action(
            "log_event",
            event_type="compliance_check",
            details={"transaction_id": "txn-123"}
        )

        assert result.success

    async def test_in_rule_context(self, harness):
        """Test plugin within rule evaluation."""
        context = MockContext(
            transaction={"id": "txn-123", "amount": 5000},
            jurisdiction="US"
        )

        result = await harness.evaluate_with_plugin(
            rule_dsl="""
                rule test_rule {
                    when {
                        let check = plugin.my_plugin.check_entity(
                            entity_id: transaction.id
                        )
                        check.match == true
                    }
                    then {
                        flag_for_review()
                    }
                }
            """,
            context=context
        )

        assert result.status == "compliant"  # No match in mock
```

### Building and Packaging

```bash
# Build plugin package
pact plugin build

# Validate plugin manifest
pact plugin validate

# Test plugin locally
pact plugin test

# Package for distribution
pact plugin package --output dist/my-plugin-1.0.0.tar.gz

# Publish to registry
pact plugin publish dist/my-plugin-1.0.0.tar.gz
```

## Plugin Security

### Sandbox Restrictions

Plugins run in an isolated sandbox with restricted capabilities:

```yaml
# Security configuration
sandbox:
  # Network restrictions
  network:
    enabled: true
    allowedHosts:
      - "api.trusted-service.com"
      - "*.internal.company.com"
    deniedPorts:
      - 22
      - 23
    maxConnections: 10
    timeoutMs: 30000

  # File system restrictions
  filesystem:
    enabled: true
    readPaths:
      - /data/plugins/${PLUGIN_NAME}
      - /config/shared
    writePaths:
      - /tmp/plugins/${PLUGIN_NAME}
    maxFileSize: 10Mi

  # Resource limits
  resources:
    maxCpu: 0.5
    maxMemory: 256Mi
    maxExecutionTime: 30s

  # Syscall restrictions
  syscalls:
    allow:
      - read
      - write
      - open
      - close
      - socket
      - connect
    deny:
      - fork
      - exec
      - ptrace
```

### Input Validation

```python
from pact.plugins import InputValidator
from pydantic import BaseModel, validator

class EntityCheckInput(BaseModel):
    """Input validation for entity check."""
    entity_id: str
    entity_type: str

    @validator('entity_id')
    def validate_entity_id(cls, v):
        if not v or len(v) > 100:
            raise ValueError('Invalid entity_id')
        return v

    @validator('entity_type')
    def validate_entity_type(cls, v):
        allowed = ['individual', 'organization', 'vessel']
        if v not in allowed:
            raise ValueError(f'entity_type must be one of {allowed}')
        return v

class SecurePlugin(DataSourcePlugin):
    @data_source(name="check_entity")
    async def check_entity(
        self,
        entity_id: str,
        entity_type: str
    ) -> DataSourceResult:
        # Validate input
        validated = EntityCheckInput(
            entity_id=entity_id,
            entity_type=entity_type
        )

        # Proceed with validated data
        return await self._do_check(validated.entity_id, validated.entity_type)
```

### Audit Logging

```python
from pact.plugins import AuditLogger

class AuditedPlugin(BasePlugin):
    def __init__(self, config: dict):
        super().__init__(config)
        self.audit = AuditLogger(plugin_name=self.metadata.name)

    @data_source(name="sensitive_check")
    async def sensitive_check(self, entity_id: str) -> DataSourceResult:
        # Log data access
        await self.audit.log(
            action="data_access",
            resource="external_database",
            details={
                "entity_id": entity_id,
                "query_type": "sanctions_check"
            }
        )

        result = await self._perform_check(entity_id)

        # Log result (without sensitive data)
        await self.audit.log(
            action="check_complete",
            resource="external_database",
            details={
                "entity_id": entity_id,
                "result": "match" if result.data.get("match") else "no_match"
            }
        )

        return result
```

## Best Practices

### Performance

1. **Caching**: Cache frequently accessed data
2. **Connection pooling**: Reuse connections to external services
3. **Async operations**: Use async for all I/O operations
4. **Batching**: Batch multiple requests when possible

### Reliability

1. **Timeout handling**: Always set and respect timeouts
2. **Retry logic**: Implement exponential backoff for retries
3. **Circuit breaker**: Use circuit breakers for external dependencies
4. **Graceful degradation**: Handle failures gracefully

### Security

1. **Input validation**: Validate all inputs
2. **Secrets management**: Never hardcode secrets
3. **Least privilege**: Request minimal permissions
4. **Audit logging**: Log all sensitive operations

### Maintainability

1. **Documentation**: Document all exported functions
2. **Versioning**: Follow semantic versioning
3. **Testing**: Write comprehensive tests
4. **Changelog**: Maintain a changelog

## Troubleshooting

### Common Issues

**Plugin fails to load**
```bash
# Check plugin logs
pact plugin logs my-plugin

# Validate manifest
pact plugin validate my-plugin

# Check dependencies
pact plugin check-deps my-plugin
```

**Network access denied**
```yaml
# Check sandbox network configuration
sandbox:
  network:
    allowedHosts:
      - "api.required-service.com"  # Add required host
```

**Timeout errors**
```python
# Increase timeout in decorator
@data_source(name="slow_check", timeout=60)
async def slow_check(self, ...):
    ...
```

## Related Documentation

- [Plugin Framework Architecture](../components/plugin-framework.md)
- [Rule Authoring Guide](./rule-authoring.md)
- [Security Model](../security/security-model.md)
- [SDK Reference](../api-reference/sdk-reference.md)
