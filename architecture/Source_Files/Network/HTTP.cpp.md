# Source_Files/Network/HTTP.cpp

## File Purpose
Implements a lightweight HTTP client wrapper around libcurl for GET and POST requests. Provides conditional support for HTTPS with configurable certificate verification, used by the game engine for network communication (e.g., metaserver interaction).

## Core Responsibilities
- Initialize libcurl at engine startup via `Init()`
- Execute HTTP GET requests and capture response bodies
- Execute HTTP POST requests with URL-encoded form parameters
- Handle response streaming via a write callback mechanism
- Support HTTPS with verification control from network preferences
- Degrade gracefully when libcurl is unavailable (stub implementations)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `HTTPClient` | class | Main HTTP client interface defined in HTTP.h; wraps libcurl operations |
| `parameter_map` | typedef | `std::map<std::string, std::string>` for POST form parameters |
| `CURL` | struct (from libcurl) | Opaque curl handle managed via `std::shared_ptr` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `network_preferences->verify_https` | bool | global (extern in preferences.h) | Controls HTTPS certificate verification for all HTTP requests |
| `escape()` | static function | file-static | URL-encodes individual POST parameter keys and values |

## Key Functions / Methods

### HTTPClient::Init()
- **Signature:** `static void Init()`
- **Purpose:** One-time initialization of libcurl global state.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `curl_global_init(CURL_GLOBAL_ALL)`; must be called before any Get/Post operations.
- **Calls:** `curl_global_init()` (libcurl)
- **Notes:** When `HAVE_CURL` is undefined, this is a no-op stub.

### HTTPClient::WriteCallback()
- **Signature:** `static size_t WriteCallback(void* buffer, size_t size, size_t nmemb, void* p)`
- **Purpose:** libcurl callback to receive response data incrementally.
- **Inputs:** `buffer` = response chunk; `size`, `nmemb` = chunk dimensions; `p` = HTTPClient instance pointer (via CURLOPT_WRITEDATA)
- **Outputs/Return:** Returns `size * nmemb` (bytes consumed) to signal success to curl.
- **Side effects:** Appends received data to the instance's `response_` member string.
- **Calls:** `std::string::append()`
- **Notes:** Cast from `void*` to `HTTPClient*`; reinterpret_cast used to convert buffer. Response accumulates across multiple callbacks.

### HTTPClient::Get()
- **Signature:** `bool Get(const std::string& url)`
- **Purpose:** Perform an HTTP GET request and return success/failure.
- **Inputs:** `url` = target URL as string
- **Outputs/Return:** `true` if curl performed successfully (CURLE_OK), `false` otherwise. Response body available via `Response()` method.
- **Side effects:** Clears prior response; initializes curl handle; sets SSL verification from `network_preferences->verify_https`; follows redirects (CURLOPT_FOLLOWLOCATION).
- **Calls:** `curl_easy_init()`, `curl_easy_setopt()` (multiple), `curl_easy_perform()`, `curl_easy_cleanup()` (via shared_ptr deleter), `curl_easy_strerror()`, `logError()`
- **Notes:** Handle lifetime managed by `std::shared_ptr<CURL>`; uses custom deleter `curl_easy_cleanup`. Logs error message on failure.

### HTTPClient::Post()
- **Signature:** `bool Post(const std::string& url, const parameter_map& parameters)`
- **Purpose:** Perform an HTTP POST request with URL-encoded form data.
- **Inputs:** `url` = target URL; `parameters` = map of key-value pairs to encode as POST body
- **Outputs/Return:** `true` on success, `false` on error. Response available via `Response()`.
- **Side effects:** Clears prior response; builds URL-encoded parameter string via `escape()`; sets CURLOPT_POST and CURLOPT_POSTFIELDS; applies HTTPS verification preference.
- **Calls:** `curl_easy_init()`, `escape()` (once per parameter), `curl_easy_setopt()` (multiple), `curl_easy_perform()`, `curl_easy_cleanup()` (via shared_ptr), `curl_easy_strerror()`, `logError()`
- **Notes:** Parameters are URL-encoded by iterating the map and calling `escape()` on each key and value; joined with "&" and "=". Handle managed by shared_ptr; logs error on failure.

### escape() [static helper]
- **Signature:** `static std::string escape(CURL* handle, const std::string& s)` (or `CURL*` unused pre-0x071504)
- **Purpose:** URL-encode a string for safe inclusion in POST data.
- **Inputs:** `handle` = curl handle (used in libcurl ΓëÑ 0x071504); `s` = string to encode
- **Outputs/Return:** URL-encoded string
- **Side effects:** Allocates memory via libcurl (freed by shared_ptr with `curl_free` deleter).
- **Calls:** `curl_easy_escape()` (newer libcurl) or `curl_escape()` (older libcurl)
- **Notes:** Conditional compilation based on `LIBCURL_VERSION_NUM`; uses shared_ptr to manage curl's returned char* with `curl_free` deleter.

## Control Flow Notes
- **Initialization:** `Init()` called once at engine startup to initialize curl globally.
- **Request phase:** `Get()` or `Post()` invoked when the engine needs to communicate with remote services (likely metaserver, based on `network_preferences` usage).
- **Response retrieval:** `Response()` accessor (in header) returns the accumulated response body.
- **Conditional compilation:** Entire libcurl integration is wrapped in `#ifdef HAVE_CURL`; when unavailable, stubs return `false`.

## External Dependencies
- **libcurl:** `curl/curl.h`, `curl/easy.h` ΓÇö HTTP client library; used for all network operations.
- **Engine utilities:** `cseries.h` ΓÇö core engine types and macros.
- **Logging:** `Logging.h` ΓÇö `logError()` macro for diagnostics.
- **Preferences:** `preferences.h` ΓÇö `network_preferences` global for `verify_https` flag.
- **HTTP header:** `HTTP.h` ΓÇö class definition and typedef for `parameter_map`.
