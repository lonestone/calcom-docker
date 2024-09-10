# Setup of the project

> :bulb: Here is the configuration of the repository and how to build and push the image. For more information about the entire configuration of the project, please see the Notion page [here](https://www.notion.so/lonestone/Projet-serveur-d-di-bde6f27f67724ec89e57999b42f8dec4).

## Steps
   
1. *Set required environment variables*
      
    List of required **Build-time variables** (⚠️ useful to build Docker image). **Set these variables in Actions secrets in the repository**.
    
    ```bash
    # Optional but maybe important for the production
    NEXT_PUBLIC_LICENSE_CONSENT=
    
    NEXT_PUBLIC_WEBAPP_URL=

    # Set this to the default value (normally optional but recommended)
    NEXT_PUBLIC_API_V2_URL=
    
    # It is highly recommended that the NEXTAUTH_SECRET must be overridden and very unique
    # Use `openssl rand -base64 32` to generate a key
    NEXTAUTH_SECRET=
    
    # Encryption key that will be used to encrypt CalDAV credentials, choose a random string, for example with `dd if=/dev/urandom bs=1K count=1 | md5sum`
    CALENDSO_ENCRYPTION_KEY=
    # [...]
    
    # Set this to '1' if you don't want Cal to collect anonymous usage
    CALCOM_TELEMETRY_DISABLED=1
    ```
    
    
    > 💡 On peut utiliser les valeurs par défaut de `.env.example` pour tester en local
        
2. Modification of the Dockerfile
    
    ```diff

    ARG NEXTAUTH_SECRET=secret
    ARG CALENDSO_ENCRYPTION_KEY=secret
    ARG MAX_OLD_SPACE_SIZE=4096
    + ARG NEXT_PUBLIC_WEBAPP_URL
    ARG NEXT_PUBLIC_API_V2_URL

    [...]
    
    RUN yarn install
    + RUN yarn prisma generate
    - RUN yarn db-deploy
    - RUN yarn --cwd packages/prisma seed-app-store

    HEALTHCHECK --interval=30s --timeout=30s --retries=5 \
    -  CMD wget --spider http://localhost:3000 || exit 1
    +  CMD wget --spider $NEXT_PUBLIC_WEBAPP_URL || exit 1
    ```
    
    Explanation :
    - Added `NEXT_PUBLIC_WEBAPP_URL` to be able to use the value given in our secrets.
    - Added `RUN yarn prisma generate`: We need this command to avoid *module not found* errors when building `@calcom/web`. This will generate a prisma client based on the `schema.prisma`
    - Removed `RUN yarn db-deploy`: We want to avoid performing migrations when building the Docker image, this allows us to avoid having to give access to the database. The migrations will be carried out when the containers are launched (using the script [`start.sh`](http://start.sh) which is the entry point of the container.
    - Removed `RUN yarn --cwd packages/prisma seed-app-store`: Basically it was to avoid a *module not found* error (but the problem was surely resolved with the addition of the generation ), but ultimately we don't even need it anymore because we don't use the database during the build.
    - Modification of the Healthcheck with the use of our url instead of `localhost:3000`
