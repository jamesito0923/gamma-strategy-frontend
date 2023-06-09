<template>
  <div class="liquidity-page main-content">
    <div class="liquidity-body">
      <div v-if="isLoading" class="liquidity-loading">
        <cv-loading overlay description="Approving" />
        <p class="liquidity-loading text">{{ loadingCopy }}</p>
      </div>
      <div class="liquidity-header">
        <h2>Visor Finance Fee Distribution</h2>
        <p>Stake GAMMA to participate in Gamma's protocol revenue</p>
      </div>

      <div class="liquidity-content">
        <cv-toast-notification
          v-if="errorNotice.visible"
          kind="info"
          class="error"
          :title="errorNotice.copy"
          :sub-title="errorNotice.description"
          :close-aria-label="errorNotice.closeAriaLabel"
          :low-contrast="false"
          @close="doCloseNotification"
        />
        <cv-toast-notification
          v-if="successNotice.visible"
          kind="success"
          class="success"
          :title="successNotice.copy"
          :sub-title="successNotice.description"
          :close-aria-label="successNotice.closeAriaLabel"
          :low-contrast="false"
          @close="doCloseNotification"
        />
        <div v-if="optionSelected == false" class="bx--form-item dropdown">
          <cv-dropdown v-model="stakingMode" value="" label="Staking Mode">
            <cv-dropdown-item value="Wallet"> From Wallet </cv-dropdown-item>
            <cv-dropdown-item value="Vault"> From Vault </cv-dropdown-item>
          </cv-dropdown>
        </div>
        <div v-if="optionSelected == true" class="bx--form-item number">
          <cv-text-input
            v-model="visrAmount"
            placeholder="0"
            label="Amount of GAMMA to stake"
            step="1"
            :mobile="false"
            :fdisabled="true"
            @change="restartApprove"
          />
          <cv-button kind="primary" size="" class="minus" @click="visrMinus()">
            -
          </cv-button>
          <cv-button kind="primary" size="" class="plus" @click="max()">
            +
          </cv-button>

          <cv-tag label="GAMMA" kind="gray" />
        </div>
      </div>

      <div class="liquidity-footer">
        <cv-button kind="secondary" @click="cancel()">
          {{ backBtnLabel }}
        </cv-button>
        <cv-button kind="primary" :disabled="!isConnected" @click="process()">
          {{ actionButton }}
        </cv-button>
      </div>
    </div>

    <cv-modal :visible="modalVisible" size="10">
      <template slot="label"> Alert </template>
      <template slot="title">
        {{ gatetitle }}
      </template>
      <template slot="content">
        <p>
          {{ gatemessage }}
        </p>
      </template>
      <template slot="secondary-button" @click="noticeFlowCancel()">
        Cancel
      </template>
      <template slot="primary-button" @click="noticeFlowProceed()">
        Proceed
      </template>
    </cv-modal>
  </div>
</template>

<script>
import { mapGetters, mapActions } from "vuex";

export default {
  name: "Visr",
  components: {},
  data() {
    return {
      core: this.$parent.$root.core,
      isLoading: true,
      visrAmount: "0",
      loadingCopy: "",
      vaultBalance: 0,
      optionSelected: false,
      stakingMode: "Wallet",
      backBtnLabel: "Cancel",
      actionButton: "Next",
      modalVisible: false,
      errorNotice: {
        visible: false,
        copy: "",
        description: "",
        closeAriaLabel: "Custom close aria label",
      },
      gatetitle: "Proceed with caution",
      gatemessage:
        "This hypervisor has not completed the audit process yet, out of an abundance of caution you should not deposit more than you are willing to lose.",
      successNotice: {
        visible: false,
        copy: "",
        description: "",
        closeAriaLabel: "Custom close aria label",
      },
    };
  },
  computed: {
    successNoticeVisible() {
      return this.successNotice.visible;
    },
    errorNoticeVisible() {
      return this.errorNotice.visible;
    },
    ...mapGetters({
      isConnected: "DataTable/getIsConnected",
      getIsInitializedPresigined: "DataTable/getIsInitializedPresigined",
      getDataTable: "DataTable/getDataTable",
    }),
  },
  watch: {
    successNoticeVisible() {
      this.errorNotice.visible = this.successNotice.visible
        ? false
        : this.errorNotice.visible;
    },
    errorNoticeVisible() {
      this.successNotice.visible = this.errorNotice.visible
        ? false
        : this.successNotice.visible;
    },
    isConnected() {
      if (!this.isConnected) {
        this.handleError("Connect Your Wallet", false);
      } else {
        this.handleSuccess("Wallet Connected");
      }
    },
  },
  async mounted() {
    /*
      if (!this.core.nonUser) {
        await this.core.preSignin();
        this.isLoading = false;
      }
      await this.core.connect(false);
    */
   
    // When presignin initializeV3 is not done.
    if (!this.getIsInitializedPresigined) {
      let dataTable = {};
      Object.assign(dataTable, this.getDataTable);
      this.$set(dataTable, "data", this.core.tableData);
      dataTable.pagination.numberOfItems = dataTable.data.length;

      this.setUpdating(true);
      this.setIsTableLoaded(false);
      await this.core.initializingDashboard(true);
      this.setIsInitializedPresigined(true);
      await this.updateLPRows();

      // Set filteredData & expandedData onto store with current state
      let expandedDataTemp = await this.core.getExpandedData(dataTable);
      this.setDataTable(dataTable);
      this.setExpandedData(expandedDataTemp);
      this.setUpdating(false);
      this.setIsTableLoaded(true);
    }
    
    this.isLoading = false;

    this.core.currentPair = "xGAMMA";

    if (
      !this.core.states.is_connected ||
      this.core.states.legacy
    ) {
      //route to dash
      this.handleError("Connect Your Wallet", false);
      return;
    }
    // Get the balance of visr and determine selection mode.

    // Get user's vault balance
    this.vaultBalance = await this.core.tokens["GAMMA"].methods
      .balanceOf(this.core.crucible._address)
      .call();

    // If user has no balance on vault then proceed on user wallet balance mode.
    if (this.vaultBalance == 0) {
      this.backBtnLabel = "Previous";
      this.optionSelected = true;
      this.stakingMode = "Wallet";
      this.backBtnLabel = "Cancel";
    }
    this.stakingMode = "Wallet";
  },
  methods: {
    ...mapActions({
      setUpdating: "DataTable/setUpdating",
      setIsTableLoaded: "DataTable/setIsTableLoaded",
      setIsInitializedPresigined: "DataTable/setIsInitializedPresigined",
      setDataTable: "DataTable/setDataTable",
      setExpandedData: "DataTable/setExpandedData",
    }),
    noticeFlowCancel() {
      // this.changeView("Loading...");
      this.modalVisible = false;
      this.$router.push({ name: "Dashboard" });
    },
    noticeFlowProceed() {
      this.modalVisible = false;
    },
    // Notification management methods
    doCloseNotification() {
      this.errorNotice.visible = false;
      this.successNotice.visible = false;
    },
    handleError(msg, timeout = true) {
      // this.successNotice.description = msg;
      // this.errorNotice.description = msg;
      this.errorNotice.copy = msg;
      this.errorNotice.visible = true;
      this.isLoading = false;
      if (timeout) {
        setTimeout(() => {
          this.errorNotice.visible = false;
        }, 5000);
      }
    },
    handleSuccess(msg) {
      // this.successNotice.description = msg;
      // this.successNotice.description = msg;
      this.successNotice.copy = msg;
      this.successNotice.visible = true;
      //       this.isLoading = false;
      setTimeout(() => {
        this.successNotice.visible = false;
      }, 5000);
    },
    async process() {
      if (this.actionButton == "Approve") {
        let allowed;
        if (this.stakingMode == "Wallet") {
          allowed = await this.core.tokens["GAMMA"].methods
            .allowance(this.core.accounts[0], this.core.rewards._address)
            .call();
        } else {
          let hash = this.core.ethers.utils.solidityKeccak256(
            ["address", "address"],
            [this.core.rewards._address, this.core.tokens["GAMMA"]._address]
          );
          allowed = await this.core.crucible.methods
            .erc20Approvals(hash)
            .call();
        }
        let unitAmount = this.core.ethers.utils.parseUnits(
          this.visrAmount,
          "18"
        );
        if (
          this.core
            .BN(unitAmount.toString())
            .lte(this.core.BN(allowed.toString()))
        ) {
          this.actionButton =
            this.stakingMode == "Vault" ? "Migrate" : "Deposit";
          return;
        }
      }
      switch (this.actionButton) {
        case "Next":
          if (!this.stakingMode) return;
          this.optionSelected = true;
          this.backBtnLabel = "Previous";
          this.actionButton = "Approve";
          break;
        case "Approve":
          if (this.stakingMode == "Wallet") await this.approveWallet();
          else await this.approveVault();
          break;
        case "Deposit":
          if (this.stakingMode == "Wallet") await this.depositFromWallet();
          else await this.depositFromUserVault();
          break;
        case "Migrate":
          if (this.stakingMode == "Wallet") await this.depositFromWallet();
          else await this.depositFromUserVault();
          break;
      }
    },
    cancel() {
      switch (this.actionButton) {
        case "Approve":
          this.backBtnLabel = "Cancel";
          this.actionButton = "Next";
          this.optionSelected = false;
          break;
        case "Deposit":
          this.backBtnLabel = "Previous";
          this.actionButton = "Approve";
          break;
        case "Migrate":
          this.backBtnLabel = "Previous";
          this.actionButton = "Approve";
          break;
        case "Next":
          this.backBtnLabel = "Cancel";
          this.actionButton = "Next";
          this.$router.push({ name: "Dashboard" });
          break;
      }
    },
    async stake() {
      let balance = await this.core.tokens["GAMMA"].methods
        .balanceOf(this.core.accounts[0])
        .call();
      if (parseInt(this.visrAmount) <= 0) return;
      let unitAmount = this.core.ethers.utils.parseUnits(this.visrAmount, "18");
      if (balance >= unitAmount) {
        this.loadingCopy = "Staking...";
        this.isLoading = true;
        const gasPrice = await this.core.gasPrice();
        await this.core.tokens["GAMMA"].methods
          .transfer(this.core.crucible._address, unitAmount)
          .send({ from: this.core.accounts[0], gasPrice}, (err, txHash) => {
            this.core.pollTx(
              txHash,
              "transfer",
              async (block) => {
                this.isLoading = false;
                this.core.msg = {
                  type: "success",
                  copy: "Success",
                  description:
                    "Staked " + this.visrAmount + " GAMMA in block " + block,
                };
                clearInterval(this.core.polls["transfer"]);
                this.changeView("Loading...");
              },
              (err) => {
                if (err.message) {
                  err = err.message;
                }
                this.isLoading = false;
                this.core.msg = {
                  type: "error",
                  copy: "Error",
                  description: err,
                };
                clearInterval(this.core.polls["transfer"]);
                this.changeView("Loading...");
              }
            );
          })
          .on("error", (err) => {
            if (err.message) {
              err = err.message;
            }
            this.isLoading = false;
            this.core.msg = {
              type: "error",
              copy: "Error",
              description: err,
            };
            clearInterval(this.core.polls["transfer"]);
            this.changeView("Loading...");
          });
      } else {
        this.isLoading = false;
        this.core.msg = {
          type: "error",
          copy: "Error",
          description: "Insufficient Wallet Balance",
        };
        this.changeView("Loading...");
      }
    },
    async depositFromWallet() {
      let unitAmount = this.core.ethers.utils.parseUnits(this.visrAmount, "18");
      this.isLoading = true;
      this.loadingCopy = "";
      this.stakingMode == "Vault" ? "Migrating..." : "Depositing...";
      const gasPrice = await this.core.gasPrice();
      //      console.log(gasPrice);
      await this.core.rewards.methods
        .enter(unitAmount)
        .send({ from: this.core.accounts[0], gasPrice}, (err, txHash) => {
          this.core.pollTx(
            txHash,
            "transfer",
            (block) => {
              this.handleSuccess("Successfully Deposited");
              setTimeout(() => {
                this.isLoading = false;
                this.changeView("Loading...");
              }, 3000);
            },
            (err) => {
              this.isLoading = false;
              this.handleError("Something went wrong on depositing.");
            }
          );
        })
        .on("error", (err) => {
          this.isLoading = false;
          this.handleError("Something went wrong on depositing.");
        });
    },
    async approveWallet() {
      let balance = await this.core.tokens["GAMMA"].methods
        .balanceOf(this.core.accounts[0])
        .call();
      if (parseInt(this.visrAmount) <= 0) return;
      let unitAmount = this.core.ethers.utils.parseUnits(this.visrAmount, "18");
      if (this.core.BN(balance).gte(this.core.BN(unitAmount.toString()))) {
        // if (balance >= unitAmount) {
        this.loadingCopy = "Approving...";
        this.isLoading = true;
        // Approving the particular amount on hypervisor address
        let units = "100000000000000000000000000";
        const gasPrice = await this.core.gasPrice();
        //        console.log(gasPrice);
        await this.core.tokens["GAMMA"].methods
          //.approve(this.core.rewards._address, unitAmount)
          .approve(this.core.rewards._address, units)
          .send({ from: this.core.accounts[0], gasPrice}, (err, txHash) => {
            this.core.pollTx(
              txHash,
              "transfer",
              async (block) => {
                this.handleSuccess("Successfully Approved");
                this.actionButton =
                  this.stakingMode == "Vault" ? "Migrate" : "Deposit";
                this.isLoading = false;
              },
              (err) => {
                this.isLoading = false;
                this.handleError("Something went wrong on approving.");
              }
            );
          })
          .on("error", (err) => {
            //console.log("ERROR ", err);
            this.handleError("Something went wrong on approving.");
            this.isLoading = false;
          });
      } else {
        this.isLoading = false;
        this.handleError("Insufficient Wallet Balance");
      }
    },
    async depositFromUserVault() {
      let unitAmount = this.core.ethers.utils.parseUnits(this.visrAmount, "18");
      this.isLoading = true;
      this.loadingCopy = "Migrating...";
      const gasPrice = await this.core.gasPrice();
      //      console.log(gasPrice);
      await this.core.rewards.methods
        .deposit(
          unitAmount,
          this.core.crucible._address,
          this.core.crucible._address
        )
        .send({ from: this.core.accounts[0], gasPrice}, (err, txHash) => {
          this.core.pollTx(
            txHash,
            "transfer",
            (block) => {
              this.handleSuccess("Successfully deposit");
              setTimeout(() => {
                this.isLoading = false;
                this.changeView("Loading...");
              }, 3000);
            },
            (err) => {
              this.isLoading = false;
              this.handleSuccess("Something went wrong on depositing");
              this.changeView("Loading...");
            }
          );
        })
        .on("error", (err) => {
          //console.log("ERROR ", err);
          this.isLoading = false;
          this.handleSuccess("Something went wrong on depositing");
        });
    },
    async approveVault() {
      // use dummy hypervisor address for a while
      // Parse the unitAmount with the decimal of token
      let unitAmount = this.core.ethers.utils.parseUnits(this.visrAmount, "18");
      // Check amount validation
      if (
        this.core.BN(this.vaultBalance).gte(this.core.BN(unitAmount.toString()))
      ) {
        // GAMMA token approve to allow hypervisor to approve from their wallet
        this.isLoading = true;
        this.loadingCopy = "Approving Transfer ERC20...";
        const gasPrice = await this.core.gasPrice();
        //        console.log(gasPrice);
        await this.core.crucible.methods
          .approveTransferERC20(
            this.core.tokens["GAMMA"]._address,
            // dummyHyperVisorAddress
            this.core.rewards._address,
            unitAmount
          )
          .send({ from: this.core.accounts[0], gasPrice}, (err, txHash) => {
            this.core.pollTx(
              txHash,
              "transfer",
              async (block) => {
                this.handleSuccess("Successfully Approved");
                this.actionButton = "Migrate";
                this.isLoading = false;
              },
              (err) => {
                this.isLoading = false;
                this.handleError("Something went wrong on approving");
              }
            );
          })
          .on("error", (err) => {
            //console.log("ERROR ", err);
            this.isLoading = false;
            this.handleError("Something went wrong on approving");
          });
      } else {
        this.isLoading = false;
        this.handleError("Insufficient Vault Balance");
      }
    },
    changeView(copy) {
      this.loadingCopy = copy;
      this.isLoading = true;

      setTimeout(() => {
        this.$router.push({ name: "Dashboard" });
      }, 3000);
    },
    prettyPrintAmount(amount) {
      let amountFormatted = this.core.web3.utils
        .toBN(amount)
        .div(
          this.core.web3.utils.toBN(10).pow(this.core.web3.utils.toBN("18"))
        );
      return amountFormatted.toString();
    },
    async max() {
      let balance;
      if (this.stakingMode == "Wallet") {
        balance = await this.core.tokens["GAMMA"].methods
          .balanceOf(this.core.accounts[0])
          .call();
      } else {
        balance = this.vaultBalance;
      }
      let dec = 18; //await this.core.tokens[aludel].methods.decimals().call();
      let hundreds = this.core.web3.utils
        .toBN(dec)
        .sub(this.core.web3.utils.toBN("2"));
      let balLockedFormatted = this.core.web3.utils
        .toBN(balance)
        .div(this.core.web3.utils.toBN(10).pow(this.core.web3.utils.toBN(dec)));
      let balLockedFormattedDex = this.core.web3.utils
        .toBN(balance)
        .div(
          this.core.web3.utils.toBN(10).pow(this.core.web3.utils.toBN(hundreds))
        )
        .toNumber()
        .toString()
        .slice(-2);
      this.visrAmount =
        balLockedFormatted.toString() + "." + balLockedFormattedDex.toString();
      // this.visrAmount = this.prettyPrintAmount(balance);
    },
    restartApprove() {
      this.actionButton = "Approve";
      this.backBtnLabel = "Previous";
    },
    visrMinus() {
      this.visrAmount = `${--this.visrAmount}`;
    },
    visrPlus() {
      this.visrAmount = `${++this.visrAmount}`;
    },
  },
};
</script>

<style lang="scss">
@import "../assets/css/main-theme.scss";
</style>
