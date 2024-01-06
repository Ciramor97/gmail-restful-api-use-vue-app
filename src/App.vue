<template>
  <div class="flex flex-col space-y-3">
    <h1 class="text-3xl font-bold mb-10">Gmail API Quickstart</h1>

    <!-- Add buttons to initiate auth sequence and sign out -->
    <button
      ref="authorizeButton"
      @click="handleAuthClick"
      class="bg-green-400 py-2 rounded-lg text-white font-bold"
    >
      Authorize
    </button>
    <button
      ref="signoutButton"
      @click="handleSignoutClick"
      class="bg-green-400 py-2 rounded-lg text-white font-bold"
    >
      Sign Out
    </button>

    <pre id="content" style="white-space: pre-wrap"></pre>
  </div>
</template>

<script setup>
import { ref, onMounted } from "vue";
const CLIENT_ID = import.meta.env.VITE_APP_API_CLIENT_ID;
const API_KEY = import.meta.env.VITE_APP_API_API_KEY;
const DISCOVERY_DOC =
  "https://www.googleapis.com/discovery/v1/apis/gmail/v1/rest";
const SCOPES = "https://www.googleapis.com/auth/gmail.readonly";

let tokenClient;
let gapiInited = false;
let gisInited = false;

const authorizeButton = ref(null);
const signoutButton = ref(null);

onMounted(() => {
  console.log(
    "onmounted",
    import.meta.env.VITE_APP_API_CLIENT_ID,
    import.meta.env.VITE_APP_API_API_KEY
  );
  // Load Google API and Identity Services libraries
  const gapiScript = document.createElement("script");
  gapiScript.src = "https://apis.google.com/js/api.js";
  gapiScript.async = true;
  gapiScript.defer = true;
  gapiScript.onload = gapiLoaded;
  document.head.appendChild(gapiScript);

  const gisScript = document.createElement("script");
  gisScript.src = "https://accounts.google.com/gsi/client";
  gisScript.async = true;
  gisScript.defer = true;
  gisScript.onload = gisLoaded;
  document.head.appendChild(gisScript);
});

function gapiLoaded() {
  gapi.load("client", initializeGapiClient);
}

async function initializeGapiClient() {
  await gapi.client.init({
    apiKey: API_KEY,
    discoveryDocs: [DISCOVERY_DOC],
  });
  gapiInited = true;
  maybeEnableButtons();
}

function gisLoaded() {
  tokenClient = google.accounts.oauth2.initTokenClient({
    client_id: CLIENT_ID,
    scope: SCOPES,
    callback: "",
  });
  gisInited = true;
  maybeEnableButtons();
}

function maybeEnableButtons() {
  if (gapiInited && gisInited) {
    authorizeButton.value.style.visibility = "visible";
  }
}

function handleAuthClick() {
  tokenClient.callback = async (resp) => {
    if (resp.error !== undefined) {
      throw resp;
    }
    signoutButton.value.style.visibility = "visible";
    authorizeButton.value.innerText = "Refresh";
    await listMessages();
  };

  if (gapi.client.getToken() === null) {
    tokenClient.requestAccessToken({ prompt: "consent" });
  } else {
    tokenClient.requestAccessToken({ prompt: "" });
  }
}

function handleSignoutClick() {
  const token = gapi.client.getToken();
  if (token !== null) {
    google.accounts.oauth2.revoke(token.access_token);
    gapi.client.setToken("");
    document.getElementById("content").innerText = "";
    authorizeButton.value.innerText = "Authorize";
    signoutButton.value.style.visibility = "hidden";
  }
}

async function listMessages() {
  let response;
  try {
    response = await gapi.client.gmail.users.messages.list({
      userId: "me",
    });
    console.log("messages===", response.result.messages);
  } catch (err) {
    console.log("error", err);
    return;
  }
  const messages = response.result.messages;
  if (!messages || messages.length == 0) {
    document.getElementById("content").innerText = "No messages found.";
    return;
  }

  // const test = await gapi.client.gmail.users.messages.get({
  //   userId: "me",
  //   id: "18cbc581d52cdba4",
  // });
  // console.log("test===", test.result.payload.parts);
  messages.forEach(async (message) => {
    fetchMessageContent(message.id);
  });

  // console.log("output==", messages);
  // document.getElementById("content").innerText = output;
}

let data = [];

function decodeBase64(s) {
  var e = {},
    i,
    b = 0,
    c,
    x,
    l = 0,
    a,
    r = "",
    w = String.fromCharCode,
    L = s.length;
  var A = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/";
  for (i = 0; i < 64; i++) {
    e[A.charAt(i)] = i;
  }
  for (x = 0; x < L; x++) {
    c = e[s.charAt(x)];
    b = (b << 6) + c;
    l += 6;
    while (l >= 8) {
      ((a = (b >>> (l -= 8)) & 0xff) || x < L - 2) && (r += w(a));
    }
  }
  return r;
}
async function fetchMessageContent(messageId) {
  try {
    data = await gapi.client.gmail.users.messages.get({
      userId: "me",
      id: messageId,
    });
    if (data.result && data.result.payload && data.result.payload.parts) {
      const payloadParts = data.result.payload.parts;
      // console.log("body==", body);
      if (payloadParts.length > 0) {
        payloadParts.forEach((payload) => {
          if (payload && payload.body.data) {
            const finalData = payload.body.data;
            // console.log("finaleData", finalData);
            const outputReplace1 = finalData.replace("-", "+");
            const outputReplace2 = outputReplace1.replace("_", "/");
            const ok = decodeBase64(outputReplace2.replace(/^[^,]+,/, ""));

            // const out3 = decodeURIComponent(
            //   escape(
            //     atob(
            //       outputReplace2
            //         .replace("-", "")
            //         .replace("+", "")
            //         .replace("/", "")
            //     )
            //   )
            // );
            // const outputReplace3 = outputReplace2.replace(/\s/g, "");

            // const decodeValue = decodeURIComponent(atob(outputReplace3));
            // console.log("outputReplace3", decodeValue);
            console.log("outputReplace3", ok);
          }
        });
      }
    }
  } catch (error) {
    console.log(error);
  }

  // .replace(/\s/g, "")
  // const body = data.payload.parts;
  // console.log("body==", body);
  // const decodedBody = atob(body);
  // const messageDetails = {
  //   Subject: getHeader(data.value, "Subject"),
  //   From: getHeader(data.value, "From"),
  //   Date: getHeader(data.value, "Date"),
  //   Content: decodedBody,
  // };
  // console.log("messageDetails==", messageDetails);
  // document.getElementById("content").innerText = output;
}

function getHeader(message, headerName) {
  const headers = message.payload.headers;
  const header = headers.find((header) => header.name === headerName);
  return header ? header.value : "N/A";
}
</script>
