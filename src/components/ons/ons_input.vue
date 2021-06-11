<template>
  <div class="ons-input">
    <ONSInputForm
      ref="form"
      :submit-label="submit_label"
      :disable-name="updating || renewing"
      :updating="updating"
      :renewing="renewing"
      :show-clear-button="updating || renewing"
      :disable-submit-button="disable_submit_button"
      @onSubmit="onSubmit"
      @onClear="onClear"
    />
    <q-inner-loading :showing="ons_status.sending" :dark="theme == 'dark'">
      <q-spinner color="primary" size="30" />
    </q-inner-loading>
  </div>
</template>

<script>
import { mapState } from "vuex";
import ONSInputForm from "./ons_input_form";
import WalletPassword from "src/mixins/wallet_password";
const objectAssignDeep = require("object-assign-deep");

export default {
  name: "ONSInput",
  components: {
    ONSInputForm
  },
  mixins: [WalletPassword],
  data() {
    return {
      updating: false,
      renewing: false
    };
  },
  computed: mapState({
    theme: state => state.gateway.app.config.appearance.theme,
    ons_status: state => state.gateway.ons_status,
    unlocked_balance: state => state.gateway.wallet.info.unlocked_balance,
    disable_submit_button() {
      const minBalance = this.updating ? 0.05 : 7.1;
      return this.unlocked_balance < minBalance * 1e9;
    },
    submit_label() {
      let label = "buttons.purchase";
      if (this.updating) {
        label = "buttons.update";
      } else if (this.renewing) {
        label = "buttons.renew";
      }
      return this.$t(label);
    }
  }),

  watch: {
    ons_status: {
      handler(val, old) {
        if (val.code == old.code) return;
        const { code, message } = val;
        switch (code) {
          case 0:
            this.$q.notify({
              type: "positive",
              timeout: 1000,
              message
            });
            this.$refs.form.reset();
            this.renewing = false;
            this.updating = false;
            break;
          case -1:
            this.$q.notify({
              type: "negative",
              timeout: 3000,
              message
            });
            break;
        }
      },
      deep: true
    }
  },
  methods: {
    startUpdating(record) {
      this.$refs.form.setRecord(record);
      this.updating = true;
    },
    startRenewing(record) {
      this.renewing = true;
      // set the type such that we default to one year
      let renewRecord = {
        ...record,
        type: "lokinet_1y"
      };
      this.$refs.form.setRecord(renewRecord);
    },
    onSubmit(record) {
      if (this.updating) {
        this.update(record);
      } else if (this.renewing) {
        this.renew(record);
      } else {
        this.purchase(record);
      }
    },
    onClear() {
      this.$refs.form.reset();
      this.updating = false;
      this.renewing = false;
    },
    async update(record) {
      const updatedRecord = {
        ...record,
        value: record.value,
        owner: record.owner,
        backup_owner: record.backup_owner
      };

      let passwordDialog = await this.showPasswordConfirmation({
        title: this.$t("dialog.onsUpdate.title"),
        noPasswordMessage: this.$t("dialog.onsUpdate.message"),
        ok: {
          label: this.$t("dialog.onsUpdate.ok"),
          color: "primary"
        },
        color: "#1F1C47"
      });
      passwordDialog
        .onOk(password => {
          // if no password set
          password = password || "";
          this.$store.commit("gateway/set_ons_status", {
            code: 1,
            message: "Sending transaction",
            sending: true
          });
          const ons = objectAssignDeep.noMutate(updatedRecord, {
            password
          });
          this.$gateway.send("wallet", "update_ons_mapping", ons);
        })
        .onDismiss(() => {})
        .onCancel(() => {});
    },
    async purchase(record) {
      let passwordDialog = await this.showPasswordConfirmation({
        title: this.$t("dialog.purchase.title"),
        noPasswordMessage: this.$t("dialog.purchase.message"),
        ok: {
          label: this.$t("dialog.purchase.ok"),
          color: "primary"
        }
      });
      passwordDialog
        .onOk(password => {
          // if no password set
          password = password || "";
          this.$store.commit("gateway/set_ons_status", {
            code: 1,
            message: "Sending transaction",
            sending: true
          });
          const ons = objectAssignDeep.noMutate(record, {
            password
          });
          this.$gateway.send("wallet", "purchase_ons", ons);
        })
        .onDismiss(() => {})
        .onCancel(() => {});
    },
    async renew(record) {
      let passwordDialog = await this.showPasswordConfirmation({
        title: this.$t("dialog.renew.title"),
        noPasswordMessage: this.$t("dialog.renew.message"),
        ok: {
          label: this.$t("dialog.renew.ok"),
          color: "primary"
        },
        dark: this.theme == "dark",
        color: this.theme == "dark" ? "white" : "dark"
      });
      passwordDialog
        .onOk(password => {
          // if no password set
          password = password || "";
          this.$store.commit("gateway/set_ons_status", {
            code: 1,
            message: "Sending renew mapping transaction",
            sending: true
          });
          const params = {
            type: record.type,
            name: record.name,
            password
          };
          this.$gateway.send("wallet", "ons_renew_mapping", params);
        })
        .onDismiss(() => {})
        .onCancel(() => {});
    }
  }
};
</script>

<style lang="scss"></style>
