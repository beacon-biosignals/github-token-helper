# github-token-helper

A [Git credential helper](https://git-scm.com/docs/gitcredentials) which assists with handling GitHub authentication using Personal Access Tokens (PAT).

Primarily, this is used as a more secure alternative to using `insteadOf` inside of a Dockerfile. For example take the following Dockerfile snippet which uses `--build-arg` to pass in a secret:

```Dockerfile
ARG GITHUB_TOKEN
RUN git config --global url."https://${GITHUB_TOKEN}:@github.com/".insteadOf "https://github.com/"

# Private repo
RUN git clone https://github.com/MyOrg/PrivateRepo.git

# Prevent leaking `GITHUB_TOKEN` into the container's runtime environment.
RUN git config --global --remove-section url."https://${GITHUB_TOKEN}:@github.com/"
```

The above works but using `--build-arg` to pass in the secret is bad as this information is embedded in the image and is easily visible by using `docker history <image>`.

A better approach is to use [`docker build --secret`](https://docs.docker.com/develop/develop-images/build_enhancements/#new-docker-build-secret-information) which can be secure if used correctly. Take the following example:

```Dockerfile
RUN --mount=type=secret,id=github_token \
    git config --global url."https://$(cat /run/secrets/github_token):@github.com/".insteadOf "https://github.com/"

# Private repo
RUN git clone https://github.com/MyOrg/PrivateRepo.git

# Prevent leaking `GITHUB_TOKEN` into the container's runtime environment.
RUN --mount=type=secret,id=github_token \
    git config --global --remove-section url."https://$(cat /run/secrets/github_token):@github.com/"
```

The secret information should no longer be leaked via the image history but since Docker uses layer caching the secret is still available in some of the image's layers.

A solution to this problem is to only use the secret within the `RUN` instruction for which it is needed. We could call `git config` use it and then unset the value all in the same instruction. However, if we need to use the secret over multiple `RUN` instructions we will need to either duplicate the logic or refactor the logic into a re-usable script. One variation on the re-usable script would be to make use of a [custom git credential helper](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage#_a_custom_credential_cache) which can make use of the secret in when the secret is mounted but avoid embedding the secret in any layer. For example:

```Dockerfile
# Install github-token-helper
RUN curl -sSL https://raw.githubusercontent.com/beacon-biosignals/github-token-helper/v0.1.1/github-token-helper -o $HOME/.github-token-helper && \
    chmod +x $HOME/.github-token-helper && \
    git config --global credential.https://github.com.helper "$HOME/.github-token-helper -f /run/secrets/github_token"

# Private repo
RUN --mount=type=secret,id=github_token \
    git clone https://github.com/MyOrg/PrivateRepo.git
```

## Installation

The basic installation requires the script to present on your system and registered as a [custom helper](https://git-scm.com/docs/gitcredentials#_custom_helpers):

```bash
curl -sSL https://raw.githubusercontent.com/beacon-biosignals/github-token-helper/v0.1.1/github-token-helper -o $HOME/.github-token-helper
chmod +x $HOME/github-token-helper
git config --global credential.https://github.com.helper "$HOME/github-token-helper -f /run/secrets/github_token -e GITHUB_TOKEN"
```

## Configuration

The `github-token-helper` accepts the following options:

- `--file` / `-f`: Specify a file containing the PAT. Used with `docker build --secret`.
- `--env` / `-e`: The name of an environmental variable which contains the PAT to use. Should not be used with Docker's `--build-arg` to avoid credential leaking but can be useful for running the container interactively.

## Testing

You can test the behavior of this script by running the following and entering key/value
pairs or just pressing enter twice:

```bash
echo 's3cre7' > mysecret.txt
./github-token-helper -f mysecret.txt get
```

When installed you can test the behavior of this this credential helper (and any other helpers you have installed) via:

```bash
echo -e "protocol=https\nhost=github.com\nusername=x" | git credential fill
```

The above is useful in validating the credentials used by the current system's setup.
