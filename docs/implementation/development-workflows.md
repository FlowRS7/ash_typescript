# Development Workflows and Best Practices

## Overview

This guide covers development workflows, testing patterns, debugging techniques, and extension points for AshTypescript development.

## Runtime Introspection with Tidewave MCP

**NEW**: This project now has Tidewave MCP enabled for powerful runtime introspection during development.

### Tidewave-Enhanced Development Workflow

**✅ Modern approach using Tidewave tools:**

```elixir
# 1. Explore the codebase
mcp__tidewave__project_eval("exports(AshTypescript.Rpc.Pipeline)")

# 2. Test specific functions in context
mcp__tidewave__project_eval("""
alias AshTypescript.Rpc.RequestedFieldsProcessor

fields = ["id", "title", %{"user" => ["name"]}]
atomized = RequestedFieldsProcessor.atomize_requested_fields(fields)
RequestedFieldsProcessor.process(AshTypescript.Test.Todo, :read, atomized)
""")

# 3. Check configuration
mcp__tidewave__project_eval("Application.get_all_env(:ash_typescript)")

# 4. Debug issues in real-time
mcp__tidewave__project_eval("""
# Test your hypothesis immediately
conn = %Plug.Conn{} |> Plug.Conn.put_private(:ash, %{actor: nil, tenant: nil})
AshTypescript.Rpc.run_action(:ash_typescript, conn, %{
  "action" => "list_todos",
  "fields" => ["id", "title"]
})
""")
```

### When to Use Tidewave vs Tests

**Use Tidewave for:**
- ✅ Quick hypothesis testing
- ✅ Exploring function behavior
- ✅ Configuration checking
- ✅ Interactive debugging
- ✅ Understanding runtime state

**Use Tests for:**
- ✅ Permanent regression prevention
- ✅ Complex scenario validation
- ✅ CI/CD pipeline integration
- ✅ Documentation of expected behavior

### Tidewave-First Debugging Pattern

```elixir
# 1. First, understand the current state
mcp__tidewave__project_eval("Ash.Info.domains(:ash_typescript)")

# 2. Test your assumption interactively
mcp__tidewave__project_eval("AshTypescript.Test.Todo |> Ash.Resource.Info.public_attributes() |> Enum.map(&(&1.name))")

# 3. Once you understand the issue, write a proper test
# test/debug_specific_issue_test.exs
defmodule DebugSpecificIssueTest do
  use ExUnit.Case
  # ... proper test implementation
end
```

## Development Workflows

### 1. Test-Driven Development Pattern

**PATTERN**: Create comprehensive test cases first, then implement support.

```elixir
# 1. Create test showing desired behavior
test "embedded resource calculations work" do
  params = %{
    "fields" => [
      %{"metadata" => ["category", "displayCategory", "adjustedPriority"]}
    ]
  }
  
  result = Rpc.run_action(:ash_typescript, conn, params)
  assert %{success: true, data: data} = result
  assert data["metadata"]["displayCategory"] == "urgent"
end

# 2. Run test to see failure
# 3. Implement minimum code to make test pass
# 4. Refactor and expand
```

### 2. TypeScript Validation Workflow

**PATTERN**: Always validate TypeScript compilation after changes.

```bash
# 1. Generate TypeScript types
mix test.codegen

# 2. Test compilation
cd test/ts && npm run compileGenerated

# 3. Test valid patterns
cd test/ts && npm run compileShouldPass

# 4. Test invalid patterns (should fail)
cd test/ts && npm run compileShouldFail

# 5. Run Elixir tests
mix test
```

### 3. Debug Module Pattern

**PATTERN**: Use isolated test modules for debugging complex issues.

```elixir
# Create test/debug_issue_test.exs
defmodule DebugIssueTest do
  use ExUnit.Case

  # Minimal resource for testing specific issue
  defmodule TestResource do
    use Ash.Resource, domain: nil
    
    attributes do
      uuid_primary_key :id
      attribute :test_field, :string, public?: true
    end
    
    actions do
      defaults [:create, :read, :update, :destroy]
    end
  end

  test "debug specific issue" do
    # Test the problematic function directly
    result = MyModule.problematic_function(TestResource)
    IO.inspect(result, label: "Debug result")
    assert true
  end
end
```

## Anti-Patterns and Critical Gotchas

### 1. Environment Anti-Patterns

```elixir
# ❌ WRONG - Using dev environment
mix ash_typescript.codegen
iex -S mix

# ❌ WRONG - One-off debugging commands without context
echo "Code.ensure_loaded(...)" | iex -S mix

# ✅ CORRECT - Test environment with Tidewave tools
mix test.codegen
# Use Tidewave for exploration and debugging:
mcp__tidewave__project_eval("Code.ensure_loaded(AshTypescript.Test.Todo)")

# ✅ CORRECT - Then write proper tests for permanent validation
# Write proper tests for debugging
```

### 2. Field Classification Anti-Patterns

```elixir
# ❌ WRONG - Incorrect classification order
cond do
  is_simple_attribute?(field_name, resource) -> :simple_attribute  # WRONG
  is_embedded_resource_field?(field_name, resource) -> :embedded_resource
end

# ❌ WRONG - Missing field types
def classify_field(field_name, resource) do
  cond do
    is_calculation?(field_name, resource) -> :simple_calculation
    is_simple_attribute?(field_name, resource) -> :simple_attribute
    true -> :unknown  # Missing aggregates, relationships, embedded resources
  end
end
```

### 3. Type Inference Anti-Patterns

```elixir
# ❌ WRONG - Assuming all complex calculations need fields
user_calculations =
  complex_calculations
  |> Enum.map(fn calc ->
    """
    #{calc.name}: {
      args: #{arguments_type};
      fields: string[]; // Wrong! May return primitive
    };
    """
  end)

# ❌ WRONG - Complex conditional types with never fallbacks
type BadProcessField<Resource, Field> = 
  Field extends Record<string, any>
    ? UnionToIntersection<{
        [K in keyof Field]: /* complex logic */ | never
      }[keyof Field]>
    : never; // Causes TypeScript to return 'unknown'
```

### 4. Unified Field Format Anti-Patterns

```elixir
# ❌ WRONG - Using removed calculations parameter
params = %{
  "fields" => ["id"],
  "calculations" => %{"self" => %{"args" => %{}}}
}

# ❌ WRONG - Referencing removed functions
convert_traditional_calculations_to_field_specs(calculations)
```

## Debugging Patterns

### Strategic Debug Outputs

**PATTERN**: Use strategic debug outputs for complex field processing.

```elixir
# Add to lib/ash_typescript/rpc.ex for field processing issues
IO.puts("\n=== RPC DEBUG: Field Processing ===")
IO.inspect(client_fields, label: "📥 Client field specification")
IO.inspect({select, load}, label: "🌳 Field parser output")
IO.inspect(combined_ash_load, label: "📋 Final load sent to Ash")
IO.puts("=== END Field Processing ===\n")
```

### Union Processing Debug Pattern

```elixir
# Add debug output to key transformation points
def apply_union_field_selection(value, union_member_specs, formatter) do
  IO.inspect(value, label: "Union input")
  transformed = transform_union_type_if_needed(value, formatter)
  IO.inspect(transformed, label: "Transformed union")
  IO.inspect(union_member_specs, label: "Member specs")
  # ... rest of function
end
```

### Type Inference Debug Pattern

```bash
# 1. Check generated TypeScript structure
MIX_ENV=test mix test.codegen --dry-run

# 2. Test specific TypeScript compilation
cd test/ts && npx tsc generated.ts --noEmit --strict

# 3. Test type inference with simple example
cd test/ts && npx tsc -p . --noEmit --traceResolution
```

## Common Error Patterns and Solutions

### Environment Issues

**"No domains found" Error**:
- **Cause**: Using dev environment instead of test environment
- **Solution**: Use `mix test.codegen` instead of `mix ash_typescript.codegen`

**"Module not loaded" Error**:
- **Cause**: Test resources not available in dev environment
- **Solution**: Always use test environment for development

### Union-Specific Errors

**"Failed to load %{...} as type Ash.Type.Union"**:
- **Cause**: Complex field constraints in `:map_with_tag` definition
- **Solution**: Remove constraints block, use simple type definition

**"protocol Enumerable not implemented for DateTime"**:
- **Cause**: Trying to enumerate DateTime structs in transformation
- **Solution**: Add DateTime guards in `format_map_fields/2`

### Type Inference Errors

**TypeScript returns `unknown` instead of proper types**:
- **Cause**: Complex conditional types with never fallbacks
- **Solution**: Use schema key-based classification

**Schema keys not matching between generation and usage**:
- **Cause**: Structural detection failing
- **Solution**: Use authoritative schema keys

## Testing Strategies

### Comprehensive Test Coverage

```elixir
# Test all field types
test "field classification comprehensive" do
  # Test simple attributes
  assert classify_field(:title, MyResource) == :simple_attribute
  
  # Test relationships
  assert classify_field(:user, MyResource) == :relationship
  
  # Test calculations
  assert classify_field(:calculated_field, MyResource) == :simple_calculation
  
  # Test embedded resources
  assert classify_field(:metadata, MyResource) == :embedded_resource
  
  # Test unions
  assert classify_field(:content, MyResource) == :union_type
end
```

### Field Selection Testing

```elixir
# Test field selection security
test "field selection prevents data leakage" do
  params = %{
    "fields" => ["id", "title"]  # Only request specific fields
  }
  
  result = Rpc.run_action(:get_todo, conn, params)
  data = result.data
  
  # Verify only requested fields are present
  assert Map.has_key?(data, "id")
  assert Map.has_key?(data, "title")
  refute Map.has_key?(data, "secret_field")
end
```

### TypeScript Validation Testing

```bash
# Test positive cases
cd test/ts && npm run compileShouldPass

# Test negative cases (should fail)
cd test/ts && npm run compileShouldFail

# Test generated types
cd test/ts && npm run compileGenerated
```

## Extension Points

### 1. Adding New Type Support

1. **Location**: `lib/ash_typescript/codegen.ex:get_ts_type/2`
2. **Pattern**: Add pattern match before catch-all fallback
3. **Testing**: Add cases to `test/ts_codegen_test.exs`

```elixir
# Add to get_ts_type/2 function
def get_ts_type(%{type: Ash.Type.NewType, constraints: constraints}, context) do
  case Keyword.get(constraints, :format) do
    :email -> "string"
    :phone -> "string"
    _ -> "string"
  end
end
```

### 2. Extending RPC Features

1. **DSL Extension**: Add entities to `@rpc` section
2. **Code Generation**: Update generation functions
3. **Runtime Support**: Add processing logic

### 3. Adding Field Types

1. **Detection Function**: Add `is_new_field_type?/2`
2. **Classification**: Add to `classify_field/2`
3. **Routing**: Add to `process_field_node/3`

```elixir
# Add new field type detection
def is_new_field_type?(field_name, resource) do
  # Implementation logic
end

# Add to field classification
def classify_field(field_name, resource) do
  cond do
    is_new_field_type?(field_name, resource) -> :new_field_type
    # ... existing classifications
  end
end
```

## Performance Optimization

### Type Generation Performance

- Resource detection is cached per calculation definition
- Type mapping uses efficient pattern matching
- Template generation is done once per resource

### Runtime Performance

- Field selection happens post-Ash loading (minimizes database queries)
- Recursive processing uses tail recursion where possible
- Schema key lookup is O(1) vs O(n) structural analysis

### TypeScript Compilation Performance

- Simple conditional types perform better than complex ones
- `any` fallbacks perform better than `never` fallbacks
- Recursive type depth limits prevent infinite compilation

## Best Practices

### Code Organization

```
lib/ash_typescript/
├── codegen.ex                    # Core type generation
├── rpc.ex                       # RPC processing
├── field_formatter.ex           # Field formatting
└── rpc/
    ├── codegen.ex               # RPC-specific generation
    ├── field_parser.ex          # Field parsing and classification
    └── result_processor.ex      # Result filtering and formatting
```

### Testing Organization

```
test/ash_typescript/
├── typescript_codegen_test.exs  # Basic type generation tests
├── embedded_resources_test.exs  # Embedded resource tests
└── rpc/
    ├── rpc_actions_test.exs     # Basic RPC action tests
    ├── rpc_field_calculations_test.exs  # Field calculation tests
    └── rpc_union_field_selection_test.exs  # Union field selection tests
```

### Documentation Standards

- **Comprehensive inline docs** with examples in implementation files
- **AI-focused documentation** with actionable patterns
- **Error pattern documentation** with causes and solutions
- **Performance consideration notes** for complex operations

## Critical Success Factors

1. **Environment Discipline**: Always use test environment for development
2. **Test-Driven Development**: Create comprehensive tests first
3. **TypeScript Validation**: Always validate compilation after changes
4. **Field Classification**: Understand the five field types and their routing
5. **Schema Key Authority**: Use schema keys as authoritative classifiers
6. **Unified Format**: Never use deprecated calculations parameter
7. **Performance Awareness**: Consider impact on both generation and compilation

---

**See Also**:
- [Environment Setup](environment-setup.md) - For development environment requirements
- [Type System](type-system.md) - For type inference and schema generation
- [Field Processing](field-processing.md) - For field classification patterns
- [Troubleshooting Guides](../troubleshooting/) - For specific debugging procedures