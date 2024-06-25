# scanner-cli branch

## Process

1. Navigate to the CWPP account: Administration → scanners → Connect scanner
2. Provide the following details:
    * Name: GitHubActionScanner
    * OS: Linux

3. Copy the command details: `docker run -d  registry.aquasec.com/scanner-cli:<scanner version> daemon --token <scanner user token> --host <SaaS account URL>`
    * The scanner user token and SaaS account url are the information we need

4. Edit the workflows.yml file in the Github Repo. Reference code below:

    ```yaml
    name: Docker Image CI

    on:
    push:
        branches: [ "scanner-cli" ]
    pull_request:
        branches: [ "scanner-cli" ]

    env:
    # Registry name as configured in Aqua integrations
    IMAGE_REGISTRY: matt_reg

    jobs:

    build:

        runs-on: ubuntu-latest

        steps:
        - uses: actions/checkout@v3
        - name: Build the Docker image
        run: docker build . --file Dockerfile --tag my-demo-image:${{ github.sha }}

        # The following 2 steps can be replaced building the scanner cli within the builder image
        - name: Get Aquasec commercial scanner
        run: wget --user ${{ secrets.AQUAREG_USER }} --password ${{ secrets.AQUAREG_PSWD }} https://download.aquasec.com/scanner/2022.4/scannercli

        - name: Executable permissions
        run: chmod +x scannercli

        # Scanner authenticates to the server (-H) using a token (-A) but this can be replaced with user and password auth
        # image is registered if found compliant (--register-compliant) as belonging to the final registry (--registry).
        # the --local flag indicates a locally built image not available in the registry yet.
        - name: Scan image
        run: ./scannercli scan -H ${{ secrets.AQUA_SERVER }} -n -A ${{ secrets.TOKEN }} --local --text --register-compliant --registry $IMAGE_REGISTRY my-demo-image:${{ github.sha }} --htmlfile 1.html --textfile 2.txt --jsonfile 3.json

        - name: Tag image with Registry
        run: docker tag my-demo-image:${{ github.sha }} $IMAGE_REGISTRY/my-demo-image:${{ github.sha }}

        - name: Push to Registry
        run: echo "docker push"
    ```

5. The workflow will utilise several secrets in its execution. While hard-coding is supported, it is strongly discouraged. Navigate to Repository → Settings → Security → Secrets → Actions
6. Add the following:
    * `AQUAREG_USER`
    * `AQUAREG_PSWD`
        * `AQUAREG_USER` & `AQUAREG_PSWD` are Aqua credentials that should be provided by the product owner. This user needs to be able to download the scannercli.
    * `AQUA_SERVER`
    * `TOKEN`
        * `AQUA_SERVER` & `TOKEN` are the SaaS account url and user token from before
     * `IMAGE_REGISTRY`
        * `IMAGE_REGISTRY` the name of the aqua registry integration
7. Test the workflow with a commit; e.g. edit the README file.

## Troubleshooting

### Timeout error

If you're hitting the following error `Failed to do request: Post "***/scanmgr/twirp/scanmgr.AuthManger/AuthenticateScanner": dial tcp ************: i/o timeout`, investigate the header being given. Consider changing to https and confirming `-n` flag for no tls verification is used.
