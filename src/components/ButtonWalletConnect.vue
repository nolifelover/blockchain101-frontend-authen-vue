<template>
  <button @click="connect">
    <span v-if="!walletAddress">Connect Wallet</span>
    <span v-if="walletAddress">{{walletAddress}}</span>
  </button>
  <div v-if="isLoading" id="loadingWrapper">
    <Loading></Loading>
  </div>
  <div v-if="connected && !isLoading" id="resultWrapper">
    <p>
      ETH Balance : {{ethBalance }} <br/>
      Token Balance : {{ tokenBalance }} <br/>
      Access Token :
      <span id="accessToken" >{{accessToken}}</span>
      <br/>
      Challenge Message: {{challengeMessage}} <br/>
      Signature : {{signature}}
    </p>
  </div>
</template>

<script>
import {ethers} from "ethers";
import ERC20_ABI from "../abis/erc20.json"
import Loading from "./utils/Loading.vue";
export default {
  name: "ButtonWalletConnect",
  components: {Loading},
  data() {
    return {
      connected:false,
      walletAddress: '',
      signature: '',
      challengeMessage:'',
      accessToken: '',
      ethBalance:0,
      tokenBalance:0,
      isLoading:false,
    }
  },
  methods:{
    async getProvider(){
      if (!window.ethereum?.request) {
        alert("MetaMask is not installed!");
        return;
      }else{
        return new ethers.providers.Web3Provider(window.ethereum, "any");
      }

    },
    async connect(){
      this.isLoading = true
      if(!this.walletAddress){
        const provider = await this.getProvider();
        await provider.send("eth_requestAccounts", []);
        const signer = await provider.getSigner();
        this.walletAddress = await signer.getAddress();
        console.log("Account:", this.walletAddress);
        this.connected = true;
        await this.getTokenBalance()
        try{
          await this.fetchSigningMessage()
        }catch (e){
          console.log("%o",e)
        }finally {
          this.isLoading = false;
        }
      }else{
        await this.disconnect()
        this.isLoading = false;
      }
    },
    async disconnect() {
      this.walletAddress = '';
      this.connected = false;
    },
    async getTokenBalance(){
      const provider = await this.getProvider();
      const TOKEN_ADDR = "0x72ECb22904607Ca1a9E7eC5257Ec445d1B8608bc";
      const token = await new ethers.Contract(TOKEN_ADDR, ERC20_ABI, provider.getSigner())
      const rawTokenBalance = await token.balanceOf(this.walletAddress);
      const decimals = await token.decimals();
      this.tokenBalance = ethers.utils.formatUnits(rawTokenBalance, decimals);
      let rawEthBalance = await provider.getBalance(this.walletAddress)
      this.ethBalance = ethers.utils.formatEther(rawEthBalance)

    },
    async fetchSigningMessage(){
      const reqSignin = await fetch(
          "http://localhost:4000/v1/auth/signature/singin",
          {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              address: this.walletAddress,
            }),
          }
      );
      const resp = await reqSignin.json();
      console.log(`signing challenge reps %o`, resp);
      this.challengeMessage = resp.msg;
      const provider = await this.getProvider();
      const signer = await provider?.getSigner();
      console.log(`try to sign msg %o`, resp.msg);
      const signatureMsg = await signer?.signMessage(resp.msg);
      this.signature = (signatureMsg || "");
      const reqAuth = await fetch(
          "http://localhost:4000/v1/auth/signature/authentication",
          {
            method: "POST",
            headers: {
              "Content-Type": "application/json",
            },
            body: JSON.stringify({
              address: this.walletAddress,
              signature: signatureMsg,
            }),
          }
      );
      const respAuth = await reqAuth.json();
      console.log(`authentication resp %o`, respAuth);
      this.accessToken = respAuth.tokens.access.token
    }
  }
}
</script>

<style scoped>
 button{
   background-color: black;
   color:white;
   padding: 1.25rem 2rem;
   cursor: pointer;
   border: none;
   font-size: 1.5rem;
   border-radius: 0.5rem;
 }
 #loadingWrapper {
   margin-top: 5vh;
 }
 #resultWrapper{
   margin-top: 5vh;
 }
 #accessToken{
   font-size: 9px;
 }

</style>
