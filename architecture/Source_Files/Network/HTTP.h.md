# Source_Files/Network/HTTP.h

## File Purpose
Defines a lightweight HTTP client class for making GET and POST requests. Wraps underlying HTTP library functionality (likely libcurl) to provide simple request/response handling for game network communication.

## Core Responsibilities
- Provide HTTP GET and POST request methods
- Manage HTTP response data accumulation and retrieval
- Initialize HTTP library state at startup
- Abstract HTTP library details from game code

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| HTTPClient | class | Main HTTP client interface |
| parameter_map | typedef | Key-value map for POST parameters (std::map<std::string, std::string>) |

## Global / File-Static State
None (file header only; implementation details unknown).

## Key Functions / Methods

### Init (static)
- Signature: `static void Init()`
- Purpose: Initialize HTTP library state (e.g., curl_global_init for libcurl)
- Inputs: None
- Outputs/Return: None
- Side effects: Global HTTP library initialization
- Calls: Not visible in header
- Notes: Must be called before any HTTPClient instance usage

### Get
- Signature: `bool Get(const std::string& url)`
- Purpose: Perform an HTTP GET request and store response
- Inputs: URL string
- Outputs/Return: Boolean success/failure
- Side effects: Populates `response_` member with server response
- Calls: Likely curl_easy_perform (inferred)
- Notes: WriteCallback is invoked during execution to accumulate response body

### Post
- Signature: `bool Post(const std::string& url, const parameter_map& parameters)`
- Purpose: Perform an HTTP POST request with form parameters
- Inputs: URL and parameter key-value pairs
- Outputs/Return: Boolean success/failure
- Side effects: Populates `response_` member with server response
- Calls: Likely curl_easy_perform (inferred)
- Notes: Parameters encoded as form data

### Response
- Signature: `std::string Response()`
- Purpose: Retrieve accumulated HTTP response body
- Inputs: None
- Outputs/Return: Response string (empty if no request made or request failed)
- Side effects: None
- Notes: Valid only after successful Get/Post call

### WriteCallback (static)
- Signature: `static size_t WriteCallback(void* buffer, size_t size, size_t nmemb, void* userp)`
- Purpose: Curl write callback; appends response chunks to userp (HTTPClient instance)
- Inputs: Buffer with data, size/count pair, userp context pointer
- Outputs/Return: Bytes processed
- Side effects: Accumulates data into response_ via userp pointer
- Notes: Standard libcurl callback signature; userp should cast to HTTPClient*

## Control Flow Notes
Presumed to be called during game initialization (Init) and on-demand for network operations (Get/Post), e.g., downloading patches, leaderboard updates, or matchmaking requests.

## External Dependencies
- `<map>` ΓÇö std::map for parameter storage
- `<string>` ΓÇö std::string for URLs and responses
- **libcurl** (inferred) ΓÇö HTTP library; WriteCallback signature and static Init suggest curl usage
