# Errors

| Code | Reason                | Description                                                            |
| ---- | --------------------- | ---------------------------------------------------------------------- |
| 400  | Bad Request           | Your request is invalid.                                               |
| 401  | Unauthorized          | Apikey / secret / HMAC signature incorrect                             |
| 403  | Forbidden             | You are authenticated, but unauthorized to access this endpoint.       |
| 404  | Not Found             | Invalid endpoint.                                                      |
| 405  | Method Not Allowed    | Invalid method for endpoint eg POST instead of GET.                    |
| 422  | Unprocessible Entity  | The server understood the request, but the data provided is incorrect. |
| 429  | Too Many Requests     | You're sending too many requests too quickly. Slow down!               |
| 500  | Internal Server Error | We had a problem with our server. Try again later.                     |
| 503  | Service Unavailable   | We're temporarily offline for maintenance. Please try again later.     |
