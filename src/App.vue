<script>
// https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser#msal-basics
import { PublicClientApplication } from "@azure/msal-browser";
import axios from "axios";

const msalConfig = {
  // https://azuread.github.io/microsoft-authentication-library-for-js/ref/types/_azure_msal_browser.Configuration.html
  // https://github.com/AzureAD/microsoft-authentication-library-for-js/blob/dev/lib/msal-browser/docs/configuration.md
  auth: {
    clientId: "a61d496c-f773-4e7a-99b6-01fcbd361df7",
    authority:
      "https://login.microsoftonline.com/30ffb211-3192-4dc7-98a1-990e9e8af9c2",
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
        .get("http://localhost:8080/auth/user", {
          headers: {
            // Authorization: "Bearer ".concat(this.auth_token.accessToken),
            Authorization: "Bearer ".concat(this.access_token.idToken),
          },
        })
        .then((response) => (this.info = response));
    },
    async login() {
      console.log(123);
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
      tokenRequest.scopes = ["a61d496c-f773-4e7a-99b6-01fcbd361df7/.default"];

      await msalInstance
        .acquireTokenPopup(tokenRequest)
        .then((tokenResponse) => {
          console.log(3321);
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
    <br />
    <br />
    <div id="uinfo" style="width: 500px">
      {{ JSON.stringify(auth_token.account, "\r\n", 4) }}
    </div>
  </div>

  <!-- <RouterView /> -->
</template>

<script></script>

<style scoped></style>
