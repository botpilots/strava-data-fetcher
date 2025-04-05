# Strava Data Fetcher

This repository contains a GitHub Actions workflow to fetch data from Strava and commit it to a `data/activities.json` file.

## Workflow In Brief

The workflow runs on a schedule (every 6 hours) and uses your Strava API credentials (via GitHub Secrets) to fetch recent activities. 

It then commits the fetched activity data directly into the repository as `data/activities.json`. 

It also maintains a log (`data/fetch_log.txt`) and checksum file (`data/last_checksum.txt`).

## Setup

To enable this workflow, you need to add the following secrets in its GitHub repository settings:

1.  Go to your repository on GitHub.
2.  Navigate to `Settings` > `Secrets and variables` > `Actions`.
3.  Click `New repository secret` for each of the following:
    *   `STRAVA_CLIENT_ID`: Your Strava application's Client ID.
    *   `STRAVA_CLIENT_SECRET`: Your Strava application's Client Secret.
    *   `STRAVA_REFRESH_TOKEN`: Your Strava Refresh Token (obtained initially as described below).

**Obtaining Your Initial `STRAVA_REFRESH_TOKEN`:**

The workflow requires a long-lived `refresh_token` to operate. You need to perform the initial Strava OAuth2 authorization flow *once* manually to get this token.

1.  **Authorization:**
    *   Construct and visit an authorization URL like this in your browser (replace `YOUR_CLIENT_ID` and ensure `redirect_uri` matches your Strava app configuration - `http://localhost` is usually fine for this manual step):
      ```
      https://www.strava.com/oauth/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=http://localhost&response_type=code&approval_prompt=auto&scope=activity:read
      ```
    *   Log in to Strava (if needed) and click "Authorize".
    *   Your browser will be redirected to your `redirect_uri` (e.g., `http://localhost/?state=&code=YOUR_CODE_HERE&scope=read,activity:read`). Copy the value of the `code` parameter from the URL in your browser's address bar.

2.  **Exchange Code using Script:**
    *  Run this curl with your values, AUTHORIZATIONCODE is the code you copied from the browser.
      ```bash
        curl -X POST https://www.strava.com/oauth/token \
        -F client_id=YOURCLIENTID \
        -F client_secret=YOURCLIENTSECRET \
        -F code=AUTHORIZATIONCODE \
        -F grant_type=authorization_code
      ```

3.  **Update GitHub Secret:**
    *   Copy the new `refresh_token` and `access_token` values sent back in the response.
    *   Go back to your GitHub repository secrets (`Settings` > `Secrets and variables` > `Actions`) and paste these values into the `STRAVA_REFRESH_TOKEN` and `STRAVA_ACCESS_TOKEN` secrets.

**Important Note on Refresh Token Lifespan:**
Strava refresh tokens are long-lived and don't have a fixed expiration date like the 6-hour access tokens. However, a refresh token **will become invalid** if:
*   A new refresh token is issued during a token refresh request (the GitHub Actions workflow *might* receive a new one but currently doesn't automatically update the secret).
*   You manually revoke the application's access in your Strava settings.

If the workflow starts failing with authorization errors, you may need to repeat the manual authorization flow to get at new valid refresh token and update the GitHub secret.

Once the secrets are configured correctly, the workflow will run automatically on its schedule or can be triggered manually.

## Local Development and Testing (Workflow Simulation)

You can test the `fetch_strava_data.yml` workflow locally before committing changes using a tool like [`act`](https://github.com/nektos/act). This simulates how the workflow would run on GitHub Actions runners using Docker.

1.  **Install `act`:** Follow the installation instructions on the [`act` repository](https://github.com/nektos/act#installation). You also need Docker installed and running.
2.  **Ensure `.secrets` File is Ready:** Make sure your `.secrets` file exists in the repository root and contains valid `STRAVA_CLIENT_ID`, `STRAVA_CLIENT_SECRET`, and `STRAVA_REFRESH_TOKEN` (you can populate the refresh token using the helper script described above).
3.  **Run the Workflow:** Execute `act` from the root of your repository. To specifically run the fetch job and provide the secrets:
    ```bash
    act -j fetch --secret-file .secrets
    ```
    `act` will download the necessary Docker image and execute the steps. Check the output for success or errors. Note that `act` simulates the workflow run but does *not* perform the actual push to your GitHub repository.
4.  **Limitations:** The local environment simulated by `act` might not be identical to the GitHub Actions environment. Always verify the final workflow behavior on GitHub Actions.
