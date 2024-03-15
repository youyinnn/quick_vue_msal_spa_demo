<script>
// https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser#msal-basics
import { PublicClientApplication } from "@azure/msal-browser";
import axios from "axios";

// personal app sso single-tenant
// const clientId = "c909cf5b-0efb-4e96-9c3f-9a7610d12d1d";

// personal app sso single-tenant 2
// const clientId = "edd81949-9b05-4b47-8aa8-c060fcc2f062";

// personal app sso multi-tenant 2
// const clientId = "09f595df-9faa-4295-a1ab-17ac9422fc3d";

// personal app sso multi-tenant 3
// const clientId = "af6a4deb-d619-4fdf-b0d8-4e58b243af63";

// simwell app multi-tenant
const clientId = "031f035f-6ca6-4e1d-a0fc-f76c8ba2906f";

// personal AD
// const authority =
//   "https://login.microsoftonline.com/30ffb211-3192-4dc7-98a1-990e9e8af9c2";

// Common for multi-tenant: https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/initialization.md#optional-configure-authority
const authority = "https://login.microsoftonline.com/common/";

const msalConfig = {
  // https://azuread.github.io/microsoft-authentication-library-for-js/ref/types/_azure_msal_browser.Configuration.html
  // https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/configuration.md
  auth: {
    clientId: clientId,
    authority: authority,
    redirectUri: "http://localhost:5173/",
  },
  cache: {
    cacheLocation: "sessionStorage", // This configures where your cache will be stored
    storeAuthStateInCookie: false, // Set this to "true" if you are having issues on IE11 or Edge
  },
};

const loginRequest = {
  scopes: ["User.Read"],
};

const tokenRequest = {
  scopes: ["User.Read", "Mail.Read"],
  forceRefresh: false, // Set this to "true" to skip a cached token and go to the server to get a new token
};

const msalInstance = new PublicClientApplication(msalConfig);
await msalInstance.initialize();

export default {
  // Properties returned from data() become reactive state
  // and will be exposed on `this`.
  data() {
    return {
      count: 0,
      auth_token: {},
      access_token: {},
      username: "No User",
      info: undefined,
      tothebacksucceed: false,
      token_var_status: 0,
    };
  },

  // Methods are functions that mutate state and trigger updates.
  // They can be bound as event handlers in templates.
  methods: {
    increment() {
      this.count++;
    },
    getUInfoFromBackEnd() {
      console.log(this.access_token.accessToken);
      axios
        .get("http://localhost:8080/user", {
          headers: {
            // Authorization: "Bearer ".concat(this.auth_token.accessToken),
            Authorization: "Bearer ".concat(this.access_token.idToken),
          },
        })
        .then((response) => {
          this.info = response.data;
          this.token_var_status = response.status;
        })
        .catch((e) => {
          console.log(e);
          this.token_var_status = e.response.status;
        });
    },
    async login() {
      await msalInstance.loginPopup(loginRequest).then((tokenResponse) => {
        console.log(tokenResponse);
        this.auth_token = tokenResponse;
        const myAccounts = msalInstance.getAllAccounts();
        console.log(myAccounts);
        this.username = myAccounts[0].username;
      });
    },
    async getToken() {
      // 1. If we don't create scope by the Expose API in the Azure APP console, you can try {app_id}/.default
      // see: https://stackoverflow.com/questions/67639910/validation-of-azure-ad-token-signature-is-invalid-the-tokens-signature-resulte
      // 2. If we want to leverage the scope managed by Azure AD, we can follow: https://stackoverflow.com/questions/76009655/using-an-azure-ad-tenant-id-and-a-valid-token-issued-for-a-app-registration
      // 3. If we want to act like a fool, using the idToken to replace the access_token and send any request to the backend is also doable, but....
      // 4. Access token acquired out of the cases from the aboved will not be varified by the backend, some ideas can be found: https://authguidance.com/azure-ad-troubleshooting/
      // so this is necessary:
      // tokenRequest.scopes = [clientId + "/.default"];

      // for multi tenant app,
      // check also: https://www.youtube.com/watch?v=NyZz1ICG7dQ
      // admin user can access without any scope

      // home tenant user can use scope /.default or via the exposed api scoped
      // tokenRequest.scopes = [clientId + "/.default"];

      // but, in order to allow external personal user get the access token, we should exposed an api
      // external persional user can only access via exposed api scoped
      tokenRequest.scopes = ["api://" + clientId + "/token"];

      await msalInstance
        .acquireTokenPopup(tokenRequest)
        .then((tokenResponse) => {
          console.log(tokenResponse);
          this.access_token = tokenResponse;
        });
    },
  },

  // Lifecycle hooks are called at different stages
  // of a component's lifecycle.
  // This function will be called when the component is mounted.
  async mounted() {
    console.log(`The initial count is ${this.count}.`);
    // use this to prevent: https://stackoverflow.com/questions/66405214/browserautherror-interaction-in-progress-interaction-is-currently-in-progress
    await msalInstance
      .handleRedirectPromise()
      .then((res) => {
        console.log(res);
      })
      .catch((err) => {
        console.error(err);
      });
  },
};
</script>

<template>
  <div>
    <div>{{ username }}</div>
    <br />
    <div>
      <button @click="login">Login</button>
      <br />
      <br />
      <button @click="getToken">Get Token</button>
    </div>
    <!-- <div class="wrapper"> -->
    <!-- <HelloWorld msg="You did it!" /> -->

    <!-- <nav>
          <RouterLink to="/">Home</RouterLink>
          <RouterLink to="/about">About</RouterLink>
        </nav> -->
    <!-- </div> -->
    <br />
    <a @click="getUInfoFromBackEnd">To back End</a>
    <span> Backend token verification status: {{ token_var_status }}</span>
    <br />
    <br />
    <hr />
    <br />
    <pre id="token" style="width: 500px">
      Token Account:
      <br />
      {{ JSON.stringify(auth_token.account, undefined, 2) }}
    </pre>
    <br />

    <hr />
    <br />
    <br />
    <pre id="uinfo" style="width: 500px">
      U Info From Backend:
      <br />
      {{ JSON.stringify(access_token.account, undefined, 2) }}
    </pre>
  </div>

  <!-- <RouterView /> -->
</template>

<script></script>

<style scoped></style>
