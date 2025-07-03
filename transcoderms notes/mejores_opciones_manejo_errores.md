# Proposal: Improved Error Handling and Command Flow in TaskController

## Context
Currently, the `TaskController` can use a global error handling middleware and custom exceptions, but these are not enforced. There are two main issues:

1. In the POST endpoint for tasks, if the command is not a cancel operation, the body does nothing. The `PostCommand` method returns `-1` to indicate that the command is not a cancel, but this value is not explicitly handled in the controller.
2. When a supra profile is not found, the code throws a generic exception:

```csharp
throw new Exception($"Error. Supra profile '{supraName}' not found in BBDD.");
```

Currently, the controller can either throw exceptions (to be handled by middleware) or return HTTP status codes directly. The middleware is only relevant if the throw approach is chosen.

## Proposed Solution

### 1. Handle `PostCommand` Return Value in Controller
- If `PostCommand` returns `-1`, this should trigger the `else` branch in the controller, allowing for proper handling of non-cancel commands.
- This avoids ambiguous responses and ensures the controller logic is clear and maintainable.

#### Example:
```csharp
var result = PostCommand(command, (int)id);
if (result == -1)
{
    // Not a cancel command, execute alternative logic
    // ...
}
else
{
    return Ok(result);
}
```

### 2. Error Handling Options
You have two main approaches for error handling:

#### a) Use Custom Exceptions with Middleware
- Throw custom exceptions for specific error cases (e.g., `NotFoundException`).
- The middleware will catch these exceptions and return the appropriate HTTP status code and error message.

##### Example:
```csharp
if (/* supra profile not found */)
    throw new NotFoundException($"Supra profile '{supraName}' not found in BBDD.");
```

#### b) Return HTTP Status Codes Directly
- Instead of throwing, the controller can return `NotFound()` or other status codes directly:

```csharp
if (/* supra profile not found */)
    return NotFound($"Supra profile '{supraName}' not found in BBDD.");
```
- This approach is simpler and does not require middleware, but may lead to more repetitive code in controllers.

## Recommendation
- Explicitly handle the `-1` return value from `PostCommand` to ensure correct flow for non-cancel commands.
- Choose one error handling strategy for consistency:
    - Use custom exceptions and middleware if you want centralized error handling and response formatting.
    - Return HTTP status codes directly for simplicity in straightforward cases.
