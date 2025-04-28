# Blueprint for Terraform Test Creation

This blueprint provides a structured approach to creating Terraform tests for modules, ensuring comprehensive coverage and maintainability. **Tests should only use assertions to test logic and should never deploy actual resources.**

## Prompt to use by humans

Analyze my Terraform module and create comprehensive test files that validate its logic without resource deployment. Follow the test blueprint structure to generate a test wrapper and feature-specific .tftest.hcl files with appropriate assertions for all key module behaviors

## Agent Actions Summary

When tasked with creating tests for a Terraform module, an AI agent should:

1. **Read module files** (main.tf, variables.tf, outputs.tf) to understand:
   - Module inputs and their types/defaults
   - Module outputs
   - Key logic blocks and conditional statements
   - Resource configurations that need validation

2. **Generate test wrapper** (tests/main.tf):
   - Include necessary variables mirroring the module
   - Replicate critical local values and logic
   - Create outputs that expose internal state for assertions
   - **Important**: When using the built-in test provider, include it in required_providers

3. **For each feature or logic block**:
   - Generate a .tftest.hcl file (e.g., logging.tftest.hcl, access_control.tftest.hcl)
   - Populate with run blocks using example patterns
   - **Critical**: Include ALL required variables either in file-level variables block or in each run block
   - Add assertions that validate expected behavior

4. **Validate test files** using terraform test:
   - Run `terraform init` in the tests directory
   - Run `terraform test` to verify all tests pass
   - Fix any issues found during validation
   - **Note**: Newer versions of Terraform (1.6+) include built-in test support

## Test Structure

### Directory Layout

```
module-directory/
  ├── main.tf             # Main module code
  ├── variables.tf        # Variable definitions
  ├── outputs.tf          # Output definitions
  ├── providers.tf        # Provider requirements
  └── tests/              # Test directory
      ├── main.tf         # Test wrapper module
      ├── feature1.tftest.hcl  # Test for Feature 1
      ├── feature2.tftest.hcl  # Test for Feature 2
      └── feature3.tftest.hcl  # Test for Feature 3
```

### Test File Structure

Each `.tftest.hcl` file should follow this pattern:

```hcl
variables {
  # Default variables shared across test runs
  # IMPORTANT: Include ALL required variables here or in run blocks
  variable1 = "default_value"
  variable2 = true
}

# First test case
run "test_case_name" {
  # Always use plan, never apply - tests should validate logic only
  command = plan
  
  variables {
    # Variables specific to this test run
    variable1 = "test_specific_value"
  }
  
  # Assertions
  assert {
    condition     = output.some_output == expected_value
    error_message = "Helpful error message"
  }
}

# Additional test cases...
```

## Test Creation Process

### Step 1: Identify Test Scenarios

Identify scenarios to test based on:

- Variable inputs and edge cases
- Conditional logic in your module
- Module outputs and expected values
- Resource configuration validation
- **Practical tip**: Group related tests into dedicated test files (e.g., one file for access controls, another for environment-specific behaviors)

### Step 2: Map Module Variables and Outputs

```hcl
# Test wrapper module (tests/main.tf)
# IMPORTANT: Include the test provider in the terraform block
terraform {
  required_providers {
    mongodbatlas = {
      source = "mongodb/mongodbatlas"
    }
    # Include the test provider if using Terraform 1.6+
    test = {
      source = "terraform.io/builtin/test"
    }
  }
}

variable "var1" { type = string }
variable "var2" { type = bool }

# Local test module implementation
locals {
  # Copy critical logic from the module for testing
  result = var.var2 ? "enabled" : "disabled"
}

# Outputs for testing
output "test_result" {
  value = local.result
}
```

### Step 3: Create Test Files by Feature

Split tests by feature or functionality:

- **Input/Output Tests**: Verify output values match expectations for given inputs
- **Conditional Logic Tests**: Verify module properly handles conditional code paths
- **Edge Case Tests**: Test with null values, empty lists, etc.
- **Best practice**: Start with simple tests and progress to more complex ones

### Step 4: Write Assertions

For each test case, write assertions that validate expected behavior:

```hcl
assert {
  condition     = length(output.resources) > 0
  error_message = "Expected at least one resource to be created"
}

assert {
  condition     = output.rule_mode == "enabled"
  error_message = "Rule should be enabled when enable_rules = true"
}
```

## Advanced Test Patterns

### Testing Internal Variables or Resources

For complex modules, expose local values as outputs in your test wrapper:

```hcl
# Module under test has a local value we need to verify
locals {
  normalized_rules = [
    for rule in var.rules : {
      name    = rule.name
      enabled = true
    }
  ]
}

# Expose it as an output for testing
output "normalized_rules" {
  value = local.normalized_rules
}
```

### Mock External Dependencies

Create simplified versions of external data sources:

```hcl
# Instead of real data source
# IMPORTANT: Create realistic mock data that mimics actual responses
locals {
  # Mock data that would normally come from remote state
  mock_foundation_outputs = {
    nonsensitive_values = {
      nat_pip_canadacentral = "10.0.0.1"
      nat_pip_eastus = "10.0.0.2"
    }
  }

  mock_compute_outputs = {
    nonsensitive_values = {
      egress_public_ip_address = "10.0.0.3"
    }
  }
}
```

## Real-World Example Patterns

### Pattern 1: Testing Environment-Specific Behavior

```hcl
# From our MongoDB Atlas module tests
variables {
  project_id = "test-project-id"
  cluster_name = "test-cluster"
  mongo_db_major_version = "6.0"
  provider_name = "AZURE"
  region_name = "US_EAST"
  analytics_instance_size = "M10"
  analytics_node_count = 1
  instance_size = "M10"
  max_instance_size = "M30"
}

# Test dev environment configuration
run "validate_dev_environment_config" {
  command = plan
  
  variables {
    environment = "dev"
  }

  assert {
    condition     = output.cluster_config.backup_enabled == false
    error_message = "Backup should be disabled in dev environment"
  }

  assert {
    condition     = output.cluster_config.replication_specs.region_configs.auto_scaling.compute_enabled == false
    error_message = "Auto-scaling should be disabled in dev environment"
  }
}

# Test production environment configuration
run "validate_prod_environment_config" {
  command = plan
  
  variables {
    environment = "prod"
  }

  assert {
    condition     = output.cluster_config.backup_enabled == true
    error_message = "Backup should be enabled in prod environment"
  }
  
  assert {
    condition     = output.cloud_backup_schedule != null
    error_message = "Backup schedule should be created in prod environment"
  }
}
```

### Pattern 2: Testing Access Controls

```hcl
# From our MongoDB Atlas module tests
run "validate_default_ip_access_lists" {
  command = plan
  
  variables {
    access_list_ips = {}
    access_list_cidr = {}
  }

  assert {
    condition     = length(output.merged_ip_whitelist) == 3
    error_message = "Default IP whitelist should contain 3 infrastructure IPs"
  }

  assert {
    condition     = contains(keys(output.ip_access_list), "10.0.0.1")
    error_message = "IP access list should contain the Zscaler Canada Central IP"
  }
}

run "validate_custom_ip_access_lists" {
  command = plan
  
  variables {
    access_list_ips = {
      "192.168.1.1" = "Custom IP 1"
      "192.168.1.2" = "Custom IP 2"
    }
    access_list_cidr = {}
  }

  assert {
    condition     = length(output.merged_ip_whitelist) == 5
    error_message = "IP whitelist should contain 3 infrastructure IPs plus 2 custom IPs"
  }
}
```

### Pattern 3: Testing Backup Configuration

```hcl
# From our MongoDB Atlas module tests 
run "validate_backup_schedules" {
  command = plan
  
  variables {
    environment = "prod"
    backup_schedules = {
      daily = {
        frequency       = 1
        retention_unit  = "days"
        retention_value = 14
      }
      hourly = {
        frequency       = 6
        retention_unit  = "hours"
        retention_value = 24
      }
      weekly = {
        frequency       = 2
        retention_unit  = "weeks"
        retention_value = 6
      }
      monthly = {
        frequency       = 3
        retention_unit  = "months"
        retention_value = 24
      }
    }
  }

  assert {
    condition     = output.cloud_backup_schedule.policy_items.daily.frequency_interval == 1
    error_message = "Daily backup frequency should be set correctly"
  }

  assert {
    condition     = output.cloud_backup_schedule.policy_items.daily.retention_value == 14
    error_message = "Daily backup retention value should be set correctly"
  }
}
```

## Common Issues and Solutions

1. **Missing Required Variables Error**
   - **Issue**: `Error: No value for required variable`
   - **Solution**: Ensure ALL required variables are defined either in the file-level `variables` block or in each `run` block's `variables` section.

2. **Provider Configuration Issues**
   - **Issue**: `Error: Invalid dependency on built-in provider`
   - **Solution**: Check that your Terraform version supports the test provider and that it's correctly included in the `required_providers` block.

3. **Duplicate Resource Errors**
   - **Issue**: `Error: Duplicate provider configuration`
   - **Solution**: Remove duplicate provider blocks or configuration files that may be causing conflicts.

4. **Complex Object Testing**
   - **Issue**: Difficulty testing nested object structures
   - **Solution**: Use `output` values in the test wrapper to expose internal structures, then test specific fields.

## Best Practices

1. **Logic-Only Testing**: Tests should only validate module logic and never deploy actual resources
2. **Always Use Plan**: Always use `command = plan` instead of `command = apply` to avoid resource creation
3. **Mock External Dependencies**: Use mock data instead of real API calls or data sources
4. **Isolated Tests**: Each test should focus on a specific aspect of functionality
5. **Meaningful Names**: Use descriptive names for test files and test cases
6. **Comprehensive Coverage**: Test both happy paths and edge cases
7. **Error Messages**: Include helpful error messages that explain the expected behavior
8. **Test Real Logic**: Focus on testing the actual logic rather than just the Terraform mechanics
9. **Expose Internals**: Create outputs that expose internal module values when needed for testing
10. **Define All Variables**: Ensure all required variables are defined in either the main variables block or in run blocks

## References

- [Terraform Testing Documentation](https://developer.hashicorp.com/terraform/language/tests)
- [Terraform Tests Tutorial](https://developer.hashicorp.com/terraform/tutorials/configuration-language/test)
- [Terraform CLI Documentation](https://developer.hashicorp.com/terraform/cli/test)
- [Terraform Test CLI Command](https://developer.hashicorp.com/terraform/cli/commands/test)