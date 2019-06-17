<!-- 
Copyright 2019 OmiseGO Pte Ltd

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

     http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License. 
 -->

<template>
  <div id="app">
    <div v-if="hasWeb3">
      <div class="md-layout md-gutter">
        <div class="md-layout-item md-layout md-gutter">
          <img class="logo" src="../assets/OmiseGO_Logo.svg">
          <md-card>
            <md-card-header>
              <md-card-header-text>
                <div class="md-title">Account</div>
              </md-card-header-text>
            </md-card-header>
            <div class="md-layout-item">
              <b>Wallet Address:</b>
              {{ account.address }}
            </div>
            <div class="md-layout-item">
              <b>Rootchain Balance:</b>
              <span class="balance">{{ account.rootBalance }} wei</span>
            </div>
            <div class="md-layout-item">
              <b>Childchain Balance:</b>
              <div v-for="balance in account.childBalances">
                <span
                  class="balance"
                >{{ balance.amount }} {{ balance.symbol }} ({{ balance.currency === OmgUtil.transaction.ETH_CURRENCY ? "ETH" : balance.currency}})</span>
              </div>
            </div>
            <div class="md-layout-item">
              <md-button class="md-raised" v-on:click="refresh">Refresh Balance</md-button>
            </div>
          </md-card>
          <md-card>
            <md-card-header>
              <md-card-header-text>
                <div class="md-title">Actions</div>
              </md-card-header-text>
            </md-card-header>
            <div>
              <div class="md-layout md-gutter md-alignment-center-center">
                <md-button v-on:click="toggleDeposit" class="md-raised md-primary action">Deposit</md-button>
                <md-button v-on:click="toggleTransfer" class="md-raised md-primary action">Transfer</md-button>
                <md-button v-on:click="toggleExit" class="md-raised md-primary action">Exit</md-button>
              </div>
            </div>
          </md-card>
        </div>

        <div class="md-layout-item md-layout md-gutter">
          <md-card>
            <div v-if="transactions.length">
              <md-card-header>
                <md-card-header-text>
                  <div class="md-title">Transaction History</div>
                </md-card-header-text>
              </md-card-header>
              <md-content class="md-scrollbar">
                <div class="md-layout-item" v-for="transaction in transactions">
                  <span class="date">
                    <b>Date:</b>
                    {{ new Date(transaction.block.timestamp * 1000).toLocaleString() }}
                  </span>
                  <span class="txhash">
                    <b>Transaction Hash:</b>
                    {{ transaction.txhash }}
                  </span>
                  <div class="result" v-for="result in transaction.results">
                    <div class="txhash">
                      <b>Currency:</b>
                      {{ result.currency }}
                    </div>
                    <div class="txhash">
                      <b>Value:</b>
                      {{ result.value }}
                    </div>
                  </div>
                </div>
              </md-content>
            </div>
          </md-card>
        </div>
      </div>
      <EventLog ref="eventLog"/>

      <Deposit
        v-if="isShowDeposit"
        v-on:close="toggleDeposit()"
        v-bind:OmgUtil="OmgUtil"
        v-bind:rootChain="rootChain"
        v-bind:activeAccount="account"
        v-bind:plasmaContractAddress="plasmaContractAddress"
      />

      <Transfer
        v-if="isShowTransfer"
        v-on:close="toggleTransfer()"
        v-bind:OmgUtil="OmgUtil"
        v-bind:childChain="childChain"
        v-bind:rootChain="rootChain"
        v-bind:activeAccount="account"
      />

      <Exit
        v-if="isShowExit"
        v-on:close="toggleExit()"
        v-bind:rootChain="rootChain"
        v-bind:childChain="childChain"
        v-bind:activeAccount="account"
        v-bind:utxos="utxos"
      />
    </div>
    <div v-else class="load-wallet">
      <h2>Enable MetaMask to continue...</h2>
    </div>
  </div>
</template>

<script>
import EventLog from "./EventLog.vue";
import modal from "./Modal.vue";
import Deposit from "./Deposit.vue";
import Transfer from "./Transfer.vue";
import Exit from "./Exit.vue";
import OmgUtil from "@omisego/omg-js-util";
import EmbarkJS from "./embarkArtifacts/embarkjs";
import pify from "pify";

const web3Options = { transactionConfirmationBlocks: 1 };

export default {
  name: "app",
  components: {
    EventLog,
    modal,
    Deposit,
    Transfer,
    Exit
  },
  data() {
    return {
      hasWeb3: false,
      isShowDeposit: false,
      isShowExit: false,
      isShowTransfer: false,
      rootChain: {},
      childChain: {},
      OmgUtil: OmgUtil,
      plasmaContractAddress: "",
      utxos: [],
      transactions: [],
      // rootBalance: account.rootBalance,
      // childBalances: account.childBalances
      account: {
        rootBalance: 0,
        childBalances: []
      }
    };
  },
  mounted() {
    this.init();
  },
  methods: {
    info: function(message) {
      console.log(message);
      this.$refs.eventLog.info(message);
    },
    error: function(message) {
      console.error(message);
      this.$refs.eventLog.error(message);
    },
    init: async function() {
      try {
        await pify(EmbarkJS.onReady)();
        // EmbarkJS.onReady(async () => {

        const embarkJsWeb3Provider = EmbarkJS.Blockchain.Providers["web3"];
        if (!embarkJsWeb3Provider) {
          throw new Error(
            "web3 cannot be found. Please ensure you have the 'embarkjs-connector-web3' plugin installed in your DApp."
          );
        }
        await EmbarkJS.Plasma.init(embarkJsWeb3Provider.web3);

        this.hasWeb3 =
          EmbarkJS.Plasma.web3 &&
          ((EmbarkJS.Plasma.web3.currentProvider &&
            EmbarkJS.Plasma.web3.currentProvider.isMetaMask) ||
            (EmbarkJS.Plasma.web3.givenProvider &&
              EmbarkJS.Plasma.web3.givenProvider.isMetaMask));
        const {
          rootChain,
          childChain,
          plasmaContractAddress
        } = EmbarkJS.Plasma;
        this.rootChain = rootChain;
        this.childChain = childChain;
        this.plasmaContractAddress = plasmaContractAddress;

        this.refresh();
        // });
      } catch (err) {
        this.error(err);
      }
    },

    refresh: async function() {
      await EmbarkJS.Plasma.updateState();
      const { transactions, account, utxos } = EmbarkJS.Plasma.state;

      this.utxos = utxos;
      this.transactions = transactions;
      this.account = account;
    },

    toggleDeposit() {
      this.isShowDeposit = !this.isShowDeposit;
    },
    toggleTransfer() {
      this.isShowTransfer = !this.isShowTransfer;
    },
    toggleExit() {
      this.isShowExit = !this.isShowExit;
    }
  }
};
</script>

<style>
span.balance {
  display: block;
}

img.logo {
  height: 60px;
  margin-left: 20px;
  margin-top: 20px;
}

.md-layout-item {
  width: 100%;
  height: 100%;
  display: block;
  margin: 20px;
  content: " ";
}

.md-content {
  max-width: 600px;
  max-height: 700px;
  overflow: auto;
}

.md-card-header-text > .md-title {
  font-weight: 900;
}

.action {
  margin: 20px;
  background-color: blue;
  color: white;
  font-weight: 500;
}

.md-card {
  margin: 20px;
}

body {
  font-family: Verdana, Geneva, Tahoma, sans-serif;
  color: black;
}
h2 {
  text-align: center;
  margin: 8px;
}

.childchain-balance-header {
  font-size: 20;
  text-align: center;
  margin-bottom: 8px;
}

.transaction {
  padding: 6px;
}

.txhash {
  display: block;
}

.transaction .txhash {
  font-size: 10pt;
}

.transaction .date {
  font-size: 10pt;
}

.transaction .result {
  display: inline-block;
}

.popup-from-address {
  font-size: 10pt;
  margin-bottom: 10px;
}
.popup-input {
  text-align: right;
  margin-bottom: 10px;
}
.popup-input input {
  width: 200px;
}
.popup-input select {
  width: 200px;
}
</style>
