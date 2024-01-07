<!-- Copyright© 2024 AITrailblazer, LLC. -->

<script setup>
import { onBeforeMount, ref } from "vue";
import { TrashIcon } from "@heroicons/vue/solid";
import { Web5 } from "@web5/api"; // Decentralized web API
import Arweave from "arweave"; // Arweave SDK for interaction with Arweave network
import logo from "./assets/AITChainGuard.png";

// Reactive state variables to hold data and manage state
let web5; // Instance of Web5 connection
const myDid = ref(""); // Decentralized Identifier (DID) of the user
const personalEntries = ref([]); // List of personal data entries
const warningMessage = ref(""); // Message to display warnings or errors
let currentUserDid; // DID of the current user
let transactionId = ref(null); // ID of the transaction dispatched to Arweave
let userInput = ref(null); // Input from the user to be stored on Arweave
let retrievedData = ref(null); // Data retrieved from Arweave

// Data structure for new personal data entry
const newFullName = ref(""); // User's full name
const newContentDescription = ref(""); // Description of the personal content
let arweave = null; // Instance of Arweave connection

// Establish connection and load personal data on component mount
onBeforeMount(async () => {
  console.log("Attempting to connect to Web5...");
  try {
    // Establishing connection to Web5
    if (!web5) {
      await authenticateWithWeb5();
    }
    myDid.value = currentUserDid; // Setting the DID of the current user

    console.log("onBeforeMount: Attempting to load personal data entries...");

    // Loading personal data entries from the decentralized network
    const { records } = await web5.dwn.records.query({
      message: {
        filter: {
          schema: "http://example.org/personal-data-schema",
        },
        dateSort: "createdAscending", // Sorting the records chronologically
      },
    });

    console.log("onBeforeMount: Personal data entries loaded:", records);

    // Clear existing entries before loading new ones
    personalEntries.value = [];

    for (let record of records) {
      const data = await record.data.json();

      // Log the raw data for each record to debug
      console.log("onBeforeMount: Raw data for record:", data);

      const entry = {
        fullName: data.fullName, // make sure these match your data's actual structure
        content: data.content,
        id: record.id,
      };

      console.log("onBeforeMount: Mapping record to personalEntry: ", entry);
      personalEntries.value.push(entry);
    }

    const personalEntriesJSON = JSON.stringify(personalEntries.value);
    console.log("onBeforeMount: personalEntries: ", personalEntriesJSON);

    // Fetch and log transaction records
    console.log("onBeforeMount: Attempting to load transaction records...");
    const transactionDataRecords = await web5.dwn.records.query({
      message: {
        filter: {
          schema: "http://example.org/transaction-data-schema",
        },
        dateSort: "createdAscending",
      },
    });

    console.log(
      "onBeforeMount: Transaction records loaded:",
      transactionDataRecords.records
    );
    // Assuming the structure of transaction data is as expected and contains 'arweaveTransactionId' field
    if (transactionDataRecords.records.length > 0) {
      // Fetching the latest transaction ID from Web5 to resume any ongoing processes
      const lastTransactionData = await transactionDataRecords.records[
        transactionDataRecords.records.length - 1
      ].data.json(); // Get the last record
      transactionId.value = lastTransactionData.arweaveTransactionId; // Accessing 'arweaveTransactionId' from the transaction data
      console.log(
        `onBeforeMount: Restored Transaction ID from the last record: ${transactionId.value}`
      );
    } else {
      console.log("No transaction records found.");
    }
    // Initialize Arweave inside the function or ensure it's accessible globally
    arweave = Arweave.init({
      host: "arweave.net", // Default host for Arweave
      port: 443, // Default port for Arweave
      protocol: "https", // Secure protocol
    });
    console.log("Arweave.init", arweave);
  } catch (error) {
    console.error("Failed to connect or load data:", error);
  }
});

// Function to store transaction ID in Web5 for recovery and consistency
async function storeTransactionIdInWeb5(transactionId) {
  try {
    // Ensure web5 is initialized and connected
    if (!web5) {
      await authenticateWithWeb5();
    }

    const transactionData = {
      arweaveTransactionId: transactionId,
      // ... any other relevant data
    };

    // Create the record in DWN
    const { record } = await web5.dwn.records.create({
      data: transactionData,
      message: {
        schema: "http://example.org/transaction-data-schema", // Adjust schema as necessary
        dataFormat: "application/json",
      },
    });

    console.log(
      "storeTransactionIdInWeb5: Transaction ID stored in Web5:",
      record.id
    );
  } catch (error) {
    console.error("Failed to store transaction ID in Web5:", error);
  }
}

// Function to authenticate and connect with Web5
async function authenticateWithWeb5() {
  try {
    const web5Result = await Web5.connect();
    web5 = web5Result.web5;
    currentUserDid = web5Result.did;

    console.log(
      "Connected to Web5 and retrieved the user's DID:",
      currentUserDid
    );
    // You can now store the user's DID and use it to access their personal data
  } catch (error) {
    console.error(
      "Failed to connect to Web5 and retrieve the user's DID:",
      error
    );
  }
}
// Function to add new personal data
async function addPersonalData() {
  if (!web5) {
    await authenticateWithWeb5();
  }
  // Check if the full name or content description is empty
  if (!newFullName.value.trim() || !newContentDescription.value.trim()) {
    console.error("Full name and content description are required.");
    warningMessage.value =
      "Both Full Name and Content Description are required.";
    // Optionally, you can set an error message in your data to display to the user
    return; // Exit the function early if validation fails
  }

  try {
    const personalData = {
      fullName: newFullName.value,
      content: newContentDescription.value,
    };

    // Reset input fields
    newFullName.value = "";
    newContentDescription.value = "";

    // Create the record in DWN
    const { record } = await web5.dwn.records.create({
      data: personalData,
      message: {
        schema: "http://example.org/personal-data-schema", // Adjust schema as necessary
        dataFormat: "application/json",
      },
    });

    const data = await record.data.json();
    personalEntries.value.push({
      fullName: data.fullName,
      content: data.content,
      id: record.id,
    });
  } catch (error) {
    console.error("Failed to add personal data:", error);
    warningMessage.value = "Failed to add data. Check console for details.";
  }
}

// Function to delete personal data
async function deletePersonalData(entry) {
  try {
    if (!web5) {
      await authenticateWithWeb5();
    }
    // Remove from client-side state
    personalEntries.value = personalEntries.value.filter(
      (e) => e.id !== entry.id
    );

    // Delete the record from DWN
    await web5.dwn.records.delete({
      message: {
        recordId: entry.id,
      },
    });
  } catch (error) {
    console.error("Failed to delete personal data:", error);
  }
}

// Function to fetch data from Web5's decentralized network
async function fetchDataFromWeb5() {
  try {
    const { records } = await web5.dwn.records.query({
      message: {
        filter: {
          schema: "http://example.org/personal-data-schema",
        },
        dateSort: "createdAscending",
      },
    });

    // Assuming each record's data can be directly converted to JSON
    return Promise.all(records.map((record) => record.data.json())); // Make sure all promises resolve
  } catch (error) {
    console.error("Failed to fetch data from Web5:", error);
    return []; // Return an empty array or handle error as needed
  }
}

// Function to aggregate all data from Web5 and store it permanently on Arweave
async function fetchFromWeb5AndStoreDataPermanently() {
  const dataFromWeb5 = await fetchDataFromWeb5(); // Fetching data from Web5

  console.log(
    "fetchFromWeb5AndStoreDataPermanently: dataFromWeb5:",
    dataFromWeb5
  );

  userInput.value = JSON.stringify(dataFromWeb5); // Aggregating all records into a single JSON string

  console.log(
    `fetchFromWeb5AndStoreDataPermanently: aggregated userInput.value: ${userInput.value}`
  );

  await dispatchTransaction(); // Dispatching the data to Arweave
}

// Function to dispatch transaction to Arweave network
async function dispatchTransaction() {
  try {
    // Check for ArConnect availability and request user permissions
    if (!window.arweaveWallet) {
      throw new Error(
        "dispatchTransaction: ArConnect extension is not installed or not enabled."
      );
    }
    console.log(`dispatchTransaction: userInput.value: ${userInput.value}`);
    // Request permission from the user to connect to ArConnect
    await window.arweaveWallet.connect([
      "ACCESS_ADDRESS",
      "SIGN_TRANSACTION",
      "DISPATCH", // Add this line
    ]);

    // Creating transaction with aggregated data
    let tx = await arweave.createTransaction({ data: userInput.value });
    tx.addTag("App-Name", "AIT-ChainGuard");
    tx.addTag("Content-Type", "text/plain");
    tx.addTag("Version", "1.0.1");
    tx.addTag("Type", "post");

    // Signing and dispatching the transaction
    await arweave.transactions.sign(tx);

    // Dispatch the transaction
    let result = await window.arweaveWallet.dispatch(tx);

    // Storing the transaction ID for future reference
    transactionId.value = result.id;

    console.log(`dispatchTransaction: transactionId: ${transactionId.value}`);

    // Storing the transaction ID in Web5 for consistency and recovery
    await storeTransactionIdInWeb5(transactionId.value);
  } catch (error) {
    console.error("Error dispatching the transaction: ", error);
  }
}

// Function to read transaction from Arweave
async function readTransaction() {
  if (!transactionId.value) {
    console.error("No transaction ID to read from.");
    return;
  }
  try {
    console.log("transactionId.value", transactionId.value);

    // Fetch the transaction data using the transaction ID
    const data = await arweave.transactions.getData(transactionId.value, {
      decode: true,
      string: true,
    });
    console.log("Retrieved Data: data", data);

    // Store the retrieved data
    retrievedData.value = data;

    // Log the retrieved data
    console.log("Retrieved Data: retrievedData.value", retrievedData.value);
  } catch (error) {
    console.error("Error reading the transaction: ", error);
    retrievedData.value = "Failed to read from Arweave!";
  }
}
</script>

<template>
  <!-- Main container for the application interface -->
  <div
    class="flex flex-col items-center justify-center min-h-full px-8 py-12 sm:px-6"
  >
    <!-- Logo -->
    <img
      :src="logo"
      alt="AIT-ChainGuard Logo"
      class="mb-4"
      style="width: 100px; height: auto"
    />
    <!-- Section: Application Title and Connection Status -->
    <div class="sm:max-w-md sm:w-full mb-10">
      <!-- Title Display -->
      <h2 class="font-bold text-3xl text-center tracking-tight mb-2">
        AIT-ChainGuard Personal Data
        <!-- Title for the application -->
      </h2>
      <!-- Connection Status Indication -->
      <div class="text-center">
        <div v-if="!myDid">Connecting to Web5 . . .</div>
        <!-- Display while connecting -->
        <div v-else class="text-green-500">Web5 Connected</div>
        <!-- Display once connected -->
      </div>
    </div>

    <!-- Section: Warning or Error Messages -->
    <div v-if="warningMessage" class="text-red-500">
      {{ warningMessage }}
      <!-- Displays any warning or error message -->
    </div>

    <!-- Section: Form to Add Personal Data -->
    <section class="w-full max-w-md">
      <form @submit.prevent="addPersonalData" class="flex flex-col space-y-4">
        <!-- Input: Full Name-->
        <div class="mb-4">
          <label class="block text-gray-700 text-sm font-bold mb-2">
            Full Name <span class="text-red-500">*</span>
          </label>

          <input
            type="text"
            v-model="newFullName"
            placeholder="Full Name"
            required
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          />
        </div>

        <!-- Textarea: Content Description -->
        <div class="mb-4">
          <label class="block text-gray-700 text-sm font-bold mb-2">
            Content Description <span class="text-red-500">*</span>
          </label>
          <textarea
            rows="3"
            v-model="newContentDescription"
            placeholder="Enter personal data content"
            required
            class="shadow appearance-none border rounded w-full py-2 px-3 text-gray-700 leading-tight focus:outline-none focus:shadow-outline"
          ></textarea>
        </div>

        <!-- Button: Submit Form -->
        <button
          type="submit"
          class="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 px-4 rounded focus:outline-none focus:shadow-outline"
        >
          Add Data
        </button>
      </form>
    </section>

    <!-- Section: Display Personal Data Entries -->
    <section v-if="personalEntries.length > 0" class="w-full max-w-2xl mt-10">
      <!-- Container for personal data entries -->
      <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <ul>
          <!-- Loop through each personal entry and display -->
          <li
            v-for="entry in personalEntries"
            :key="entry.id"
            class="border-b border-gray-200 flex justify-between items-center py-3"
          >
            <!-- Display area for each entry's full name and content -->
            <div>
              <p class="text-gray-900 font-medium">{{ entry.fullName }}</p>
              <p class="text-gray-600">{{ entry.content }}</p>
            </div>

            <!-- Button: Delete Personal Data -->
            <button
              @click="deletePersonalData(entry)"
              class="text-red-500 hover:text-red-700"
            >
              <TrashIcon class="h-6 w-6" />
            </button>
          </li>
        </ul>
      </div>
    </section>

    <!-- Section: Buttons for Permanent Storage and Data Retrieval -->
    <section class="w-full max-w-2xl mt-10">
      <div class="bg-white shadow-md rounded px-8 pt-6 pb-8 mb-4">
        <div class="buttons">
          <!-- Button: Store Data Permanently on Arweave -->
          <section class="w-full max-w-2xl mt-10">
            <button
              class="btn"
              @click="fetchFromWeb5AndStoreDataPermanently"
              :disabled="personalEntries.length === 0"
            >
              Store Data Permanently on Arweave
            </button>
          </section>

          <!-- Button: Read Data from Arweave -->
          <section class="w-full max-w-2xl mt-10">
            <button
              class="btn1"
              @click="readTransaction"
              :disabled="!transactionId"
            >
              Read Data from Arweave
            </button>
          </section>
        </div>

        <!-- Display: Retrieved data from Arweave -->
        <div class="output" v-if="retrievedData">
          <p>Retrieved Data from Arweave: {{ retrievedData }}</p>
        </div>
      </div>
    </section>

    <section class="mt-4 text-gray-500 max-w-xl mx-auto">
      <h1 class="text-center text-2xl font-bold mb-4">
        AIT-ChainGuard User Interface Journey
      </h1>
      <p>
        Explore the AIT-ChainGuard application's user interface, detailing how
        users interact and manage their personal data using the decentralized
        capabilities of Web5 and Arweave.
      </p>

      <div>
        <h2 class="text-xl font-bold mb-2">First Run: Initial Data Entry</h2>
        <p>
          As users begin their journey, they are greeted with a prompt to enter
          personal data, marking the commencement of their data management
          journey. Enter your name and a personal note as a foundational step.
        </p>

        <h2 class="text-xl font-bold mb-2">Adding Data to Web5</h2>
        <p>
          Post initial data entry, users seamlessly transition to adding their
          data into the Web5 infrastructure, marking the first step in
          decentralized data management. Simply click "Add Data" to proceed.
        </p>

        <h2 class="text-xl font-bold mb-2">
          Storing Data Permanently on Arweave
        </h2>
        <p>
          Ensuring data permanence and integrity, users have the option to store
          their data on Arweave. On your first attempt, the ARConnect wallet
          will request credentials and permissions. Secure your data by
          selecting "Store Data Permanently on Arweave" and then verify the
          storage by clicking "Read Data from Arweave".
        </p>

        <h2 class="text-xl font-bold mb-2">Managing and Verifying Data</h2>
        <p>
          AIT-ChainGuard empowers users to manage their data actively. Add
          additional records, adjust existing data, or delete as needed. Each
          action reflects instantly in your personal data profile, offering a
          real-time view of your digital footprint.
        </p>

        <h2 class="text-xl font-bold mb-2">
          Second Run: Ongoing Accessibility
        </h2>
        <p>
          Returning to AIT-ChainGuard is as smooth as the first visit.
          Refreshing the browser re-engages the user with the application,
          allowing immediate access and management of stored data on Arweave.
          Continue where you left off, with all your data securely stored and
          readily accessible.
        </p>
      </div>
    </section>

    <section class="mt-4 text-gray-500 max-w-xl mx-auto">
      <h1 class="text-center text-2xl font-bold mb-4">
        Why ARConnect Wallet is Required for AIT-ChainGuard
      </h1>
      <p>
        ARConnect wallet is an essential component for AIT-ChainGuard for
        several key reasons:
      </p>
      <ul class="list-disc list-inside">
        <li>
          <strong>Decentralized Storage:</strong> AIT-ChainGuard relies on
          Arweave's decentralized data storage for immutable and permanent data
          retention, crucial for the integrity and reliability of the data it
          handles.
        </li>
        <li>
          <strong>Transaction Management:</strong> As a gateway to the Arweave
          network, ARConnect handles transactions, allowing users to upload
          data, interact with smart contracts, and perform other network
          operations.
        </li>
        <li>
          <strong>User Identity and Security:</strong> ARConnect ensures users'
          identities are securely managed, safeguarding access and interactions
          with the network and maintaining the privacy and security of users'
          data.
        </li>
        <li>
          <strong>Ease of Use:</strong> For those unfamiliar with blockchain
          technology, ARConnect provides a user-friendly interface, making it
          easier for users to manage their wallets and interact with
          AIT-ChainGuard's features.
        </li>
        <li>
          <strong>Smart Contract Interaction:</strong> If AIT-ChainGuard uses
          smart contracts, ARConnect is critical for executing and interacting
          with these contracts on the Arweave network.
        </li>
      </ul>
      <p>
        Overall, ARConnect wallet is a crucial tool for anyone looking to
        interact with the Arweave network, offering secure, user-friendly, and
        efficient access to decentralized services provided by AIT-ChainGuard.
      </p>
    </section>

    <section class="mt-4 text-gray-500 max-w-xl mx-auto">
      <h1 class="text-center text-2xl font-bold mb-4">
        ARConnect Wallet Guide
      </h1>
      <div class="mb-6">
        Need assistance with ARConnect? Visit the
        <a href="https://arconnect.io/" target="_blank" class="text-blue-500"
          >ARConnect Support</a
        >
        website for support and FAQs.
      </div>

      <div>
        <p>
          To download ArConnect, a browser extension for interacting with the
          Arweave network, follow these steps:
        </p>
        <ol class="list-decimal list-inside">
          <li>
            Open your preferred web browser (ArConnect is available for Chrome
            and Firefox).
          </li>
          <li>
            Go to the official ArConnect website:
            <a
              href="https://arconnect.io/"
              target="_blank"
              class="text-blue-500"
              >https://arconnect.io/</a
            >.
          </li>
          <li>
            On the ArConnect website, you should see download links for both
            Chrome and Firefox. Click on the link that corresponds to your
            browser. For example, if you are using Google Chrome, click on the
            "Download for Chrome" button.
          </li>
          <li>
            You will be redirected to the Chrome Web Store or Firefox Add-ons
            website (depending on your browser choice).
          </li>
          <li>
            On the Chrome Web Store or Firefox Add-ons page, click the "Add to
            Chrome" or "Add to Firefox" button, respectively.
          </li>
          <li>
            A pop-up window will appear, asking you to confirm the installation
            of the extension. Click the "Add Extension" or "Add" button to
            proceed with the installation.
          </li>
          <li>
            The extension will be downloaded and added to your browser. You
            should see the ArConnect icon in your browser's toolbar.
          </li>
          <li>
            Click on the ArConnect icon to open the extension. You may need to
            set up your wallet and grant permissions to use it with Arweave.
          </li>
          <li>
            Follow the on-screen instructions to create or import a wallet and
            connect it to the Arweave network.
          </li>
          <li>
            Once your wallet is set up and connected, you can start using
            ArConnect to interact with Arweave decentralized applications
            (dApps) and perform transactions on the Arweave network.
          </li>
          <li>
            That's it! You've successfully downloaded and installed ArConnect
            for your browser. You can now use it to manage your Arweave wallet
            and engage with the Arweave ecosystem.
          </li>
        </ol>
      </div>
    </section>
  </div>
</template>
<style>
.container {
  max-width: 500px;
  margin: auto;
  padding: 20px;
  box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
}

.input-field {
  width: 100%;
  padding: 10px;
  margin-bottom: 20px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.buttons {
  display: flex;
  justify-content: space-between;
}

.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background-color: #007bff;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
}
.btn1 {
  padding: 10px 20px;
  border: none;
  border-radius: 4px;
  background-color: #00834cd5;
  color: white;
  cursor: pointer;
  transition: background-color 0.3s;
}
.btn:disabled {
  background-color: #ccc;
  cursor: not-allowed;
}
button.btn1:disabled {
  background-color: #ccc !important; /* increased specificity and importance */
  cursor: not-allowed !important;
}

.output {
  margin-top: 20px;
  padding: 20px;
  background-color: #f9f9f9;
  border: 1px solid #eee;
}
</style>
