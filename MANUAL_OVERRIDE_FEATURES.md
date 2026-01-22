# Manual Fallback Override Features

## Overview

setup-ruby-flash provides two inputs that allow you to manually override the automatic fallback detection:

1. **`use-setup-ruby`**: Force specific versions to use ruby/setup-ruby (even if setup-ruby-flash supports them)
2. **`use-setup-ruby-flash`**: Force specific versions to use setup-ruby-flash (even if it doesn't normally support them)

These features enable benchmarking, A/B testing, and forward compatibility.

## Use Cases

### 1. Benchmarking Performance

Compare setup-ruby-flash performance against ruby/setup-ruby for the same Ruby version:

```yaml
jobs:
  benchmark:
    name: Benchmark ${{ matrix.ruby }} with ${{ matrix.setup }}
    strategy:
      matrix:
        ruby: ['3.4', '4.0']
        setup: ['flash', 'ruby']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup Ruby
        uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: ${{ matrix.ruby }}
          bundler-cache: true
          # Force ruby/setup-ruby when setup == 'ruby'
          use-setup-ruby: ${{ matrix.setup == 'ruby' && format('[''{0}'']', matrix.ruby) || '[]' }}
      
      - name: Benchmark gem installation
        run: |
          time bundle install
          # Compare times between flash and ruby setups
```

### 2. Forward Compatibility Testing

Test setup-ruby-flash with versions that rv doesn't support yet (but might in the future):

```yaml
jobs:
  test-future:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup Ruby head (experimental)
        uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: "head"
          bundler-cache: true
          # Attempt to use rv for head (will fail until rv supports it)
          use-setup-ruby-flash: "['head']"
        continue-on-error: true
      
      - name: Test with cutting edge Ruby
        if: success()
        run: bundle exec rake test
```

### 3. A/B Testing in CI

Run the same tests with both setups to ensure compatibility:

```yaml
jobs:
  test:
    strategy:
      matrix:
        ruby: ['3.4']
        setup: ['flash', 'ruby']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: ${{ matrix.ruby }}
          bundler-cache: true
          use-setup-ruby: ${{ matrix.setup == 'ruby' && '["3.4"]' || '[]' }}
      
      - run: bundle exec rake test
```

### 4. Gradual Migration Testing

Test specific versions with setup-ruby while migrating:

```yaml
jobs:
  test:
    strategy:
      matrix:
        ruby: ['3.2', '3.3', '3.4']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: ${{ matrix.ruby }}
          bundler-cache: true
          # Keep 3.2 using setup-ruby during migration
          use-setup-ruby: "['3.2']"
      
      - run: bundle exec rake test
```

## Input Format

lBoth inputs accept JSON array format with single or double quotes:

### Valid Formats

```yaml
# Single version
use-setup-ruby: "['3.4']"
use-setup-ruby: '["3.4"]'

# Multiple versions
use-setup-ruby: "['3.4', '4.0']"
use-setup-ruby: '["3.4", "4.0"]'

# Empty (no override)
use-setup-ruby: "[]"
use-setup-ruby: ""

# Dynamic from matrix
use-setup-ruby: ${{ matrix.setup == 'ruby' && format('[''{0}'']', matrix.ruby) || '[]' }}
```

## Precedence Rules

The override inputs have the following precedence (highest to lowest):

1. **`use-setup-ruby-flash`** - Highest priority, forces flash path
2. **`use-setup-ruby`** - Second priority, forces fallback
3. **Automatic detection** - Default behavior based on version/implementation

### Example: Conflicting Overrides

```yaml
- uses: appraisal-rb/setup-ruby-flash@v1
  with:
    ruby-version: "3.4"
    use-setup-ruby: "['3.4']"        # Would force fallback
    use-setup-ruby-flash: "['3.4']"  # Takes precedence - uses flash
```

Result: Uses setup-ruby-flash (because `use-setup-ruby-flash` has higher priority)

## Behavior Details

### use-setup-ruby

**Purpose**: Force specific Ruby versions to use ruby/setup-ruby

**When to use**:
- Benchmarking performance differences
- A/B testing for compatibility
- Keeping specific versions on setup-ruby during migration
- Testing that your app works with both setups

**Example behaviors**:

| Ruby Version | Normally | With `use-setup-ruby: "['3.4']"` |
|--------------|----------|-----------------------------------|
| 3.4          | Flash ⚡  | Fallback 🔄                       |
| 4.0          | Flash ⚡  | Flash ⚡                           |
| 3.1          | Fallback 🔄 | Fallback 🔄                    |

**Notice message**: `Ruby version '3.4' forced to use ruby/setup-ruby via use-setup-ruby input.`

### use-setup-ruby-flash

**Purpose**: Force specific Ruby versions to attempt using setup-ruby-flash

**When to use**:
- Testing future rv support (e.g., head, jruby)
- Forward compatibility preparation
- Experimental features

**Warning**: Will fail if rv doesn't actually support the version!

**Example behaviors**:

| Ruby Version | Normally | With `use-setup-ruby-flash: "['head']"` |
|--------------|----------|------------------------------------------|
| head         | Fallback 🔄 | Flash ⚡ (may fail!)                   |
| 3.4          | Flash ⚡  | Flash ⚡                                  |
| 3.1          | Fallback 🔄 | Flash ⚡ (may fail!)                   |

**Notice message**: `Ruby version 'head' forced to use setup-ruby-flash via use-setup-ruby-flash input.`

## Implementation Details

### Detection Logic

```bash
# 1. Check use-setup-ruby-flash (highest priority)
if version in use-setup-ruby-flash list:
    use flash path
    exit

# 2. Check use-setup-ruby (second priority)
if version in use-setup-ruby list:
    use fallback path
    exit

# 3. Automatic detection (default)
if version not in allowlist (3.2, 3.3, 3.4, 4.0):
    use fallback path
else:
    use flash path
```

### Parsing Logic

The inputs are parsed using shell string matching:

```bash
USE_FLASH_LIST='["3.4", "head"]'
if echo "$USE_FLASH_LIST" | grep -q "\"$RUBY_VERSION\""; then
    # Version found in list
fi
```

Supports both single and double quotes in JSON arrays.

## Common Patterns

### Pattern 1: Benchmark Matrix

```yaml
strategy:
  matrix:
    ruby: ['3.4']
    setup: ['flash', 'ruby']
steps:
  - uses: appraisal-rb/setup-ruby-flash@v1
    with:
      ruby-version: ${{ matrix.ruby }}
      bundler-cache: true
      use-setup-ruby: ${{ matrix.setup == 'ruby' && format('[''{0}'']', matrix.ruby) || '[]' }}
```

### Pattern 2: Future-Proof Head Testing

```yaml
- uses: appraisal-rb/setup-ruby-flash@v1
  with:
    ruby-version: "head"
    bundler-cache: true
    use-setup-ruby-flash: head
  continue-on-error: true  # Don't fail if rv doesn't support it yet
```

### Pattern 3: Selective Override

```yaml
- uses: appraisal-rb/setup-ruby-flash@v1
  with:
    ruby-version: ${{ matrix.ruby }}
    bundler-cache: true
    # Only force 3.2 and 3.3 to use setup-ruby
    use-setup-ruby: ['3.2', '3.3']
```

### Pattern 4: Environment-Based Override

```yaml
- uses: appraisal-rb/setup-ruby-flash@v1
  with:
    ruby-version: "3.4"
    bundler-cache: true
    # Use setup-ruby in staging, flash in production
    use-setup-ruby: ${{ github.ref == 'refs/heads/staging' && '3.4' || '' }}
```

## Troubleshooting

### Override Not Working

**Issue**: Version still uses automatic detection

**Solutions**:
1. Check JSON array syntax (quotes, brackets)
2. Verify version string matches exactly (e.g., "3.4" not "3.4.0")
3. Check for typos in input names

### use-setup-ruby-flash Causes Failure

**Issue**: Workflow fails when forcing unsupported version

**Solutions**:
1. Check if rv actually supports the version
2. Use `continue-on-error: true` for experimental testing
3. Remove override if not testing forward compatibility

### Both Overrides Set

**Issue**: Confused about which takes precedence

**Solution**: `use-setup-ruby-flash` always wins. Remove one of them.

## Best Practices

### DO ✅

- Use for benchmarking and testing
- Document why you're using overrides in comments
- Use `continue-on-error` with experimental `use-setup-ruby-flash`
- Keep override lists short and specific

### DON'T ❌

- Use in production without testing
- Force unsupported versions without `continue-on-error`
- Set both overrides for the same version (confusing)
- Use overrides as a permanent solution (they're for testing)

## Examples from the Wild

### Performance Comparison CI

```yaml
name: Performance Comparison
on: [push]

jobs:
  compare:
    strategy:
      matrix:
        ruby: ['3.4', '4.0']
        setup: ['flash', 'ruby']
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      
      - name: Setup Ruby (${{ matrix.setup }})
        uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: ${{ matrix.ruby }}
          bundler-cache: true
          use-setup-ruby: ${{ matrix.setup == 'ruby' && matrix.ruby || '' }}
      
      - name: Measure installation time
        run: |
          echo "::notice::Ruby ${{ matrix.ruby }} setup with ${{ matrix.setup }}"
          time bundle install
```

### Future Compatibility Check

```yaml
name: Test Future Ruby Versions
on: [schedule]

jobs:
  test-head:
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v5
      
      - name: Try Ruby head with rv
        uses: appraisal-rb/setup-ruby-flash@v1
        with:
          ruby-version: "head"
          bundler-cache: true
          use-setup-ruby-flash: head
      
      - name: Run tests
        run: bundle exec rake test
```

## Summary

The manual override features provide powerful control over which Ruby setup is used, enabling:

- **Benchmarking**: Compare performance objectively
- **Testing**: Validate compatibility with both setups
- **Forward Compatibility**: Prepare for future rv support
- **Flexibility**: Control setup behavior per version

Use these features wisely for testing and validation, while relying on automatic detection for production workflows.
