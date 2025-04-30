# Chapter 2: External API Integration (`mhq/exapi`)

Welcome back! In [Chapter 1: SQLAlchemy Data Models & Store](01_sqlalchemy_data_models___store_.md), we explored how `middleware` manages its *own* data using blueprints (Models) and a filing system manager (Store/Repositories). But what happens when our application needs information from *outside* itself, like from GitHub or GitLab?

Imagine `middleware` needs to display a list of all the software repositories your team uses on GitHub. Or maybe it needs to fetch the latest details about a specific Pull Request from GitLab. Our application doesn't magically know this information; it needs to *ask* these external platforms. This is where **External API Integration (`mhq/exapi`)** steps in.

Think of the `mhq/exapi` part of our project as the application's team of **ambassadors**. Each ambassador is specialized in talking to a specific external platform (like GitHub or GitLab). They know the platform's specific language (its API) and how to securely present credentials (like access tokens) to get the information we need.

## Meet the Ambassadors: `ApiService` Classes

The core players in `mhq/exapi` are the `ApiService` classes, such as `GithubApiService` and `GitlabApiService`.

* **`mhq/exapi/github.py`:** Contains `GithubApiService`, responsible for all communication with the GitHub API.
* **`mhq/exapi/gitlab.py`:** Contains `GitlabApiService`, responsible for all communication with the GitLab API.

These classes act like specialized translators and negotiators. To talk to GitHub, you'll use an instance of `GithubApiService`; to talk to GitLab, you'll use `GitlabApiService`.

**How do they get permission?** External platforms like GitHub require authentication. Our ambassadors need an **access token** (like a secret key or passport) to prove they have permission to request data on behalf of a user or organization. When we create an instance of an `ApiService`, we typically provide this token.

```python
# File: backend/analytics_server/mhq/exapi/github.py (Conceptual Snippet)

# Import necessary library to talk to GitHub
from github import Github
# Potentially requests library too for direct HTTP calls
import requests

# Define the ambassador for GitHub
class GithubApiService:
    # The ambassador needs the secret key (access token) to start
    def __init__(self, access_token: str):
        self._token = access_token
        # Use the token to initialize the GitHub library helper
        self._g = Github(self._token)
        # Store GitHub's base API address and auth header for raw requests
        self.base_url = "https://api.github.com"
        self.headers = {"Authorization": f"Bearer {self._token}"}

    # ... methods to talk to GitHub go here ...
```

**Explanation:**

* `class GithubApiService:`: Defines our ambassador class for GitHub.
* `__init__(self, access_token: str):`: The constructor. When we create a `GithubApiService`, we *must* give it the `access_token`.
* `self._token = access_token`: Stores the token for later use.
* `self._g = Github(self._token)`: Initializes a helper library (`PyGithub`) with the token, making many common GitHub tasks easier.
* `self.base_url`, `self.headers`: Stores info needed if we make direct web requests (using `requests`) instead of using the `PyGithub` library.

## Understanding the External Data: Models (`mhq/exapi/models/`)

When GitHub or GitLab sends us data, it comes in a specific format dictated by *them*. This format might be different from how we store similar information in our own database (our [SQLAlchemy Data Models & Store](01_sqlalchemy_data_models___store_.md)).

To handle this, `mhq/exapi` also includes **models**, but these are different from the SQLAlchemy models in Chapter 1. These models, found in `mhq/exapi/models/github.py` and `mhq/exapi/models/gitlab.py`, act as Python blueprints for the data *as it's received* from the external APIs.

They are often simple Python data classes (`@dataclass`) that just define the structure.

```python
# File: backend/analytics_server/mhq/exapi/models/gitlab.py (Simplified)
from dataclasses import dataclass
from typing import Dict, List # For type hinting

# A blueprint for how GitLab structures Repository information
@dataclass
class GitlabRepo:
    name: str          # The repository's name
    org_name: str      # The group/namespace it belongs to
    web_url: str       # The URL to view it in a browser
    idempotency_key: str # GitLab's unique ID for the repo
    # ... other relevant fields from GitLab API ...

    # A helper to create this object from the raw data GitLab sends
    def __init__(self, project: Dict):
        self.name = project.get("name")
        self.org_name = project.get("namespace", {}).get("full_path")
        self.web_url = project.get("web_url")
        self.idempotency_key = str(project.get("id"))
        # ... parse other fields ...
```

**Explanation:**

* `@dataclass`: A decorator that makes creating simple data-holding classes easier.
* `class GitlabRepo:`: Defines the structure for a GitLab repository *as GitLab sends it*.
* Fields like `name`, `org_name`, `web_url`: These directly map to the keys often found in the JSON response from the GitLab API.
* `__init__(self, project: Dict)`: This special method takes the raw dictionary received from GitLab (`project`) and populates the `GitlabRepo` object's fields.

These models help keep the external data organized and make it easier to work with in our Python code *before* we decide how (or if) to store it in our own database.

## Using the Ambassadors: Fetching GitHub Repositories

Let's return to our use case: fetching a list of GitHub repositories for a specific organization. How would another part of the application, perhaps the [Service Layer](06_service_layer_.md), use `mhq/exapi`?

1. **Get the Token:** First, it needs the GitHub access token for the organization. This might be retrieved from our database using the `CoreRepoService` from [Chapter 1: SQLAlchemy Data Models & Store](01_sqlalchemy_data_models___store_.md).
2. **Create the Ambassador:** It creates an instance of `GithubApiService` using the token.
3. **Ask for Repos:** It calls a method on the `GithubApiService` instance, like `get_repos_raw`, providing the organization's login name.

```python
# Somewhere in the Service Layer (Conceptual Example)
from mhq.exapi.github import GithubApiService

# Assume we got these from somewhere (e.g., user input, database)
org_github_login = "my-cool-org"
github_access_token = "ghp_secret_token_..." # The real token

# 1. Create the GitHub ambassador with the token
github_ambassador = GithubApiService(access_token=github_access_token)

try:
    # 2. Ask the ambassador to get the raw repository data
    # Note: GitHub API pagination starts from page 0
    page_number = 0
    repos_per_page = 30
    raw_repo_data = github_ambassador.get_repos_raw(
        org_login=org_github_login,
        page=page_number,
        per_page=repos_per_page
    )

    # 3. Now use the data!
    # raw_repo_data is typically a list of dictionaries,
    # each dictionary representing a repo as GitHub sent it.
    print(f"Found {len(raw_repo_data)} repositories:")
    for repo in raw_repo_data:
        print(f"- {repo.get('name')} ({repo.get('html_url')})")

except Exception as e:
    print(f"Oops, couldn't talk to GitHub: {e}")

```

**Explanation:**

* We import `GithubApiService`.
* We create an instance, passing the required `access_token`.
* We call the `get_repos_raw` method, specifying which organization (`org_login`) we're interested in, and how many results we want per page (`per_page`) and which page (`page`).
* The method returns a list (`raw_repo_data`). Each item in the list is a dictionary containing the raw data for one repository, directly from GitHub's API response.
* Our code can then process this list, for example, printing the name and URL of each repository.

## What Happens Under the Hood?

When `github_ambassador.get_repos_raw(...)` is called:

1. **Preparation:** The `get_repos_raw` method inside `GithubApiService` figures out the correct GitHub API endpoint (URL) to call, like `https://api.github.com/orgs/my-cool-org/repos`.
2. **Authentication:** It uses the stored `_token` to add the necessary `Authorization: Bearer ghp_secret_token_...` header to the request.
3. **Making the Call:** It uses a library like `requests` or the specific `PyGithub` library (`self._g`) to send an HTTP GET request to that URL with the authorization header and any parameters (like page size).
4. **Waiting for Response:** The code waits for GitHub's servers to process the request and send back a response.
5. **Parsing the Response:** GitHub typically replies with data in JSON format. The `ApiService` parses this JSON into Python data structures (like a list of dictionaries).
6. **Returning the Data:** The `get_repos_raw` method returns this list of dictionaries back to the part of the code that called it (our conceptual Service Layer example).

Here's a simplified diagram showing the interaction:

```mermaid
sequenceDiagram
    participant Service as Service Layer
    participant ExApiSvc as ExternalIntegrationsService
    participant GhApi as GithubApiService
    participant GitHub as GitHub API

    Service->>ExApiSvc: Request GitHub Repos for "my-cool-org"
    Note over Service, ExApiSvc: ExApiSvc retrieves token for "my-cool-org"
    ExApiSvc->>GhApi: Create GithubApiService(token)
    ExApiSvc->>GhApi: get_repos_raw("my-cool-org", page=0, per_page=30)
    GhApi->>GitHub: GET /orgs/my-cool-org/repos?page=1&per_page=30 (with Auth Header)
    Note right of GhApi: PyGithub library might handle URL mapping and sending request.
    GitHub-->>GhApi: JSON Response with list of Repos
    GhApi-->>ExApiSvc: Return List[Dict] (Raw Repo Data)
    ExApiSvc-->>Service: Return List[Dict]
```

*(Note: The `ExternalIntegrationsService` shown in the diagram (`mhq/service/external_integrations_service.py`) is often used as a convenience layer. It helps select the correct `ApiService` (GitHub or GitLab) based on configuration and fetches the necessary token before creating and using the specific `ApiService`.)*

## A Quick Look Inside `GithubApiService`

Let's peek at a *simplified* version of how `get_repos_raw` might be implemented:

```python
# File: backend/analytics_server/mhq/exapi/github.py (Simplified Method)
from github import Github # The main GitHub library
from github.GithubException import GithubException # For handling GitHub errors

class GithubApiService:
    def __init__(self, access_token: str):
        self._token = access_token
        # Initialize the library helper
        self._g = Github(self._token)
        # Define default items per page for API calls
        self.page_size = 100
        self._g.per_page = self.page_size # Tell library default page size

    # Method to get raw repository data for an organization
    def get_repos_raw(
            self, org_login: str, per_page: int = 30, page: int = 0
        ) -> list[dict]: # Expects to return a list of dictionaries
        try:
            # Find the specific organization object using the library
            organization = self._g.get_organization(org_login)

            # Ask the library to get repositories for this org
            # It handles making the actual API call behind the scenes
            # .get_page(page) fetches the specific page requested
            paginated_repos = organization.get_repos().get_page(page)

            # Extract the raw dictionary data from each repo object
            raw_data = [repo.raw_data for repo in paginated_repos]
            return raw_data

        except GithubException as e:
            # Handle potential errors from GitHub (like bad token, rate limits)
            print(f"GitHub API error: {e.status} - {e.data}")
            # Re-raise the exception so the caller knows something went wrong
            raise e
```

**Explanation:**

* It uses the initialized `self._g` (the `PyGithub` library instance).
* `self._g.get_organization(org_login)` finds the organization.
* `organization.get_repos().get_page(page)` interacts with the PyGithub library to fetch the *specific page* of repositories. The library handles constructing the API URL, adding headers, sending the request, and parsing the initial response into `Repository` objects.
* We then loop through the `Repository` objects returned by the library and access their `.raw_data` attribute (which `PyGithub` conveniently stores) to get the original dictionaries.
* Error handling (`try...except GithubException`) is crucial for dealing with network issues, invalid tokens, or API rate limits.

## Conclusion

We've seen that the `mhq/exapi` directory and its contents (`ApiService` classes and external Models) act as our application's interface to the outside world of services like GitHub and GitLab.

* **Purpose:** To communicate with external APIs securely and reliably.
* **Key Components:**
  * `GithubApiService`, `GitlabApiService`: The "ambassadors" holding access tokens and methods to call specific API endpoints.
  * External Models (`mhq/exapi/models/`): Blueprints defining the structure of data *received* from external APIs.
* **Function:** Fetches data (repos, PRs, users, etc.) from external platforms.

Now that we understand how `middleware` fetches data from external sources, how does it keep track of *how much* data it has fetched? Imagine fetching thousands of pull requests – we don't want to fetch them all every single time! We need a way to remember "I've already fetched everything up to *this* point." That's where bookmarks come in.

Next up: [Chapter 3: Bookmark Service](03_bookmark_service_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)
