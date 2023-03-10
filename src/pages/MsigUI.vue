<template>
	<a-layout-content class="content_sign">
		<ErrorBlock :error="error" :update-error="(val:string) => (error = val)" />
		<a-row>
			<a-col :xs="{ span: 24 }" :lg="{ span: 11 }">
				<h3>Partial sign transaction</h3>
				<p>
					Add a new or a partially signed
					<a
						href="https://algorand.github.io/js-algorand-sdk/interfaces/EncodedSignedTransaction.html"
						target="_blank"
						>Encoded Signed Transaction
					</a>
					in base64 Msgpack and sign it by connecting your account using web
					wallets.
				</p>
				<div class="sign_field">
					<a-textarea
						v-model:value="unsignedJson"
						placeholder="Base64 msgpack"
						:rows="20"
						allow-clear
					>
					</a-textarea>
				</div>
				<a-button type="primary" :loading="btnLoader" @click="sign"
					>SIGN</a-button
				>
				<MultisigParameters
					@getAddress="setAddresses"
					:inputBase64="unsignedJson"
				/>
			</a-col>
			<a-col :xs="{ span: 24 }" :lg="{ span: 11, offset: 2 }">
				<h3>Transaction preview</h3>
				<p>
					This is a preview of your Encoded Signed Transaction in JSON or base64
					Msgpack format. Once all required signatures are aggregated (i.e
					minimum threshold is reached), you can send the transaction using the
					<a @click="propsHomeTabChange(Tabs.TxSender)">Tx Sender</a> Tab.
				</p>
				<a-card
					class="card"
					title="Format Transaction"
					:tab-list="tabList"
					:active-tab-key="key"
					:bordered="false"
					@tabChange="(key) => onTabChange(key, 'key')"
				>
					<template #customTab="item">
						<span>
							{{ item.key }}
						</span>
					</template>
					<a-textarea
						class="text_area"
						:auto-size="{ maxRows: 22 }"
						:bordered="false"
						v-model:value="contentList[key]"
						:disabled="true"
					/>
				</a-card>
				<MultisigParameters :inputBase64="contentList.MSG_PACK" />
			</a-col>
		</a-row>
	</a-layout-content>
</template>

<script lang="ts">
import WalletStore from "@/store/WalletStore";
import algosdk, { Transaction } from "algosdk";
import { defineComponent, reactive, toRefs, ref } from "vue";
import { WalletType, contentlist, Tabs, MultisigAddr } from "@/types";
import {
	errorMessage,
	WALLET_NOT_SUPPORTED,
	NO_WALLET,
	openErrorNotificationWithIcon,
	openSuccessNotificationWithIcon,
	SIGN_SUCCESSFUL,
	tabList,
	ERROR_TITLE,
} from "@/constants";
import MultisigParameters from "@/components/multisigParameters.vue";
import {
	convertBase64ToUint8Array,
	formatJSON,
	prettifyTransaction,
	signMultisigUsingMyAlgoWallet,
	assertAddrPartOfMultisig,
	signMultisigUsingAlgosigner,
} from "@/utilities";
import ErrorBlock from "@/components/ErrorBlock.vue";

export default defineComponent({
	name: "Multisignature UI",
	components: {
		MultisigParameters,
		ErrorBlock,
	},
	props: {
		onHomeTabChange: Function,
	},
	data() {
		const walletStore = WalletStore();

		const contentList: contentlist = {
			JSON: "JSON",
			MSG_PACK: "MSG_PACK(base64)",
		};

		const state = reactive({
			signed: false,
		});

		const key = ref("tab1");

		const onTabChange = (value: string, type: string) => {
			if (type === "key") {
				key.value = value;
			}
		};

		return {
			unsignedJson: "",
			signedJson: "",
			buttonClass: "btn btn-danger",
			formControl: "form-control",
			walletStore,
			Transaction,
			tabList,
			contentList,
			...toRefs(state),
			key,
			onTabChange,
			Tabs,
			msigAddresses: [{}],
			error: "",
			btnLoader: false,
		};
	},
	methods: {
		setAddresses(value: MultisigAddr[]) {
			this.msigAddresses = value;
		},
		propsHomeTabChange(value: Tabs) {
			typeof this.onHomeTabChange === "function" && this.onHomeTabChange(value);
		},
		displayError(error: Error) {
			errorMessage(this.key);
			openErrorNotificationWithIcon(ERROR_TITLE);
			this.error = error.message;
		},
		async sign() {
			let txnBase64 = "";
			txnBase64 = this.unsignedJson;
			try {
				if (this.walletStore.walletKind) {
					this.btnLoader = true;
					await assertAddrPartOfMultisig(
						this.msigAddresses,
						this.walletStore.address
					);
					this.btnLoader = false;
					switch (this.walletStore.walletKind) {
						case WalletType.MY_ALGO: {
							let jsonObject = algosdk.decodeObj(
								convertBase64ToUint8Array(txnBase64)
							) as algosdk.SignedTransaction;
							const { base64, json } = await signMultisigUsingMyAlgoWallet(
								txnBase64,
								jsonObject.txn
							);
							this.contentList.MSG_PACK = base64;
							this.contentList.JSON = formatJSON(prettifyTransaction(json));
							this.signed = true;
							this.key = "MSG_PACK";
							openSuccessNotificationWithIcon(SIGN_SUCCESSFUL);
							break;
						}
						case WalletType.ALGOSIGNER: {
							const { base64, json } = await signMultisigUsingAlgosigner(
								txnBase64
							);
							this.contentList.MSG_PACK = base64;
							this.contentList.JSON = formatJSON(prettifyTransaction(json));
							this.signed = true;
							this.key = "MSG_PACK";
							openSuccessNotificationWithIcon(SIGN_SUCCESSFUL);
							break;
						}
						case WalletType.WALLET_CONNECT: {
							openErrorNotificationWithIcon(WALLET_NOT_SUPPORTED);
							break;
						}
						default: {
							openErrorNotificationWithIcon(NO_WALLET);
							break;
						}
					}
				} else throw Error(NO_WALLET);
			} catch (error) {
				this.btnLoader = false;
				this.displayError(error);
			}
		},
	},
});
</script>
