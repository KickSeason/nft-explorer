<template>
<div id="explorer">
	<div class="row mt-5">
		<div class="col-md-6 col-10 mx-auto text-center">
			Welcome to O3's Non fungible token (NFT) explorer and minter. This POC is to help developers get started with NFT's
			in the NEO Smart Economy. <a href="#">Learn More</a>
		</div>
	</div>
    <div class="row mt-3 mb-4">
		<div class="col-10 col-md-6 mx-auto">
			<div class="input-group mb-1">
				<input class="form-control" v-model="search_value" v-bind:class= "{'is-valid': valid_input == true, 'is-invalid': valid_input == false}"
				 placeholder="Contract or User Address" aria-label="NEO address" aria-describedby="basic-addon2">
				<div class="input-group-append">
					<button type="button" v-on:click="searchForValue" class="btn btn-primary btn-o3-primary">
						Search
					</button>
				</div>
				<div class="invalid-feedback">
					Please provide a valid contract or user address
				</div>
				<div class="valid-feedback">
					{{valid_text}}
				</div>
			</div>
		</div>
	</div>

	<div v-if="unknownError == true" class="alert alert-danger col-10 col-md-6 mx-auto">
  	Looks like the testnet might be having issues, please check back later
	</div>

	<div v-if="addressHasNoTokens == true" class="alert alert col-10 col-md-6 mx-auto text-center">
  	Looks like this address or contract doesn't have any NFT's, try and mint a new one in the <router-link to="/foundry">Token Foundry</router-link>
	</div>

	<section class="container">
		<div class="text-center success-message mb-4" v-if="totalSupply != undefined && totalSupply > 0"> 
				{{ totalSupply }} NFT's Found
		</div>

		<div class="row row-grid d-lg-flex">
			<div class="col-sm-6 col-lg-4 mb-lg-4" v-for="nft in tokens">
				<NFTCard :token_id="nft.token_id" :owner="nft.owner" :uri="nft.uri" :contract="nft.contract"></NFTCard>
			</div>
		</div>

		<div class="d-flex justify-content-center" v-if="totalSupply != undefined && total_pages > 0">
			<nav aria-label="Token Paging">
				<ul class="pagination">
					<li class="page-item paging-button"><a class="page-link" v-on:click="decrementPage">Previous</a></li>
					<li class="page-item paging-button"><a class="page-link">{{current_page + 1}}</a></li>
					<li class="page-item paging-button"><a class="page-link" v-on:click="incrementPage" >Next</a></li>
				</ul>
			</nav>
		</div>
    </section>

		
		
		<section class="container" v-if="totalSupply == undefined || totalSupply == 0"> 
				<div class="col-12 mx-auto text-center">
					<div class="landing-info-title mb-1">Don't know any NFT addresses or contracts? Check these out!</div>
					<div class="landing-info mb-1">O3 Foundry <a href="#" v-on:click="navigateToFoundry">7fe1d36ed60846975e70ec8b6fc0bef08b033107</a></div>
					<div class="landing-info mb-1">HashPuppies, the OG NFT <a href="#" v-on:click="navigateToHashPuppy">e7b2046b2412c4c7f1531ce144a73d47c3b272fe</a></div>
					<div class="landing-info mb-1">Or mint your own tokens in the O3 Foundry</div>
				</div>
	</section>
	<modal ref="modal" :title="modalTitle" :description="modalDescription" :modalAction="modalAction"></modal>
</div>
</template>

<script type="text/javascript">
	import NFTCard from './NFTCard.vue'
	import modal from './../modal.vue'

	export default {
		components: {
			NFTCard,
			modal
		},
		data: function () {
			return {
				search_value:"",
				valid_input: undefined,
				addressHasNoTokens: false,
				valid_text: "Successfully Validated Contract",
				contract_hash: "",
				address: "",
				token_id: 0,
				image_url:"",
				totalSupply: undefined,
				//Known Contracts should probably be parsed out into a server side method
				known_contracts: [ "7fe1d36ed60846975e70ec8b6fc0bef08b033107",
													"4e2d82efae9bae7e6d3ccb016ab768a607c49c10",
													"0df71b4a31a35b4dc86eeb8d062bc9d8d6235929"
													],
				tokens:[],
				current_page: 0,
				items_per_page: 9,
				total_pages: 0,

				//modal stuff
				modalTitle: "Connect to the test network",
        modalDescription: "Looks like your O3 wallet is set to the mainnet. Currently only test net is available for this app. Please change to testnet in the settings menu",
				modalAction: function() {},

				unknownError: false
			}
		}, 
		methods:{
			buildURIRequest(id, contract) {
				var uriRequest = {
						"scriptHash": contract,
  						"operation": "uri",
  						"args": [
    						{
      							"type": 'Integer',
      							"value": parseInt(id, 10)
      						}
  						],
  						"network": "TestNet"
					}
				return uriRequest
			},

			buildOwnerOfRequest(id, contract) {
				var ownerRequest = {
						"scriptHash": contract,
						"operation": "ownerOf",
						"args": [
							{
								"type": 'Integer',
								"value": parseInt(id, 10)
							}
						],
						"network": "TestNet"
					}
				return ownerRequest
			},

			buildTotalSupplyRequest() {
				var totalSupplyRequest = {
						"scriptHash": this.contract_hash,
						"operation": "totalSupply",
						"args": [],
						"network": "TestNet"
					}
				return totalSupplyRequest
			},
			
			buildGetTokensOfOwnerRequest(scriptHash) {
				var tokensOfOwnerRequest = {
					"scriptHash": scriptHash,
						"operation": "tokensOfOwner",
						"args": [
							{
								"type": 'ByteArray',
								"value": Neon.u.reverseHex(Neon.wallet.getScriptHashFromAddress(this.address))
							}, {
								"type": "Integer",
								"value": 1
							}
						],
					"network": "TestNet"
				}
				return tokensOfOwnerRequest
			},
			convertHexToString(hex) {
    			var str = '';
    			for (var i = 0; (i < hex.length && hex.substr(i, 2) !== '00'); i += 2)
        			str += String.fromCharCode(parseInt(hex.substr(i, 2), 16));
    			return str;
			},

			incrementPage() {
				console.log("Incrementing Page")
				if (this.current_page == this.total_pages - 1) {
					return
				}
				this.current_page = this.current_page + 1
				this.loadTokensForContractPage()
			},

			decrementPage() {
				console.log("Decrementing Page")
				if (this.current_page == 0) {
					return
				}
				this.current_page = this.current_page - 1
				this.loadTokensForContractPage()
			},

			navigateToFoundry() {
				this.search_value = "7fe1d36ed60846975e70ec8b6fc0bef08b033107"
				this.searchForValue()
			},

			navigateToHashPuppy() {
				this.search_value = "e7b2046b2412c4c7f1531ce144a73d47c3b272fe"
				this.searchForValue()
			},
			
			loadAllTokensForAddress() {
				this.$emit('isWaitingForDapi')

				var self = this
				var smartEcoRouter = new smartEco.SmartEcoRouter()
				smartEcoRouter.start()
				self.tokens = []
				self.current_page = 0
				self.total_pages = 0
				self.totalSupply = 0
				var counterForTotalSupply = 0
				for (var i = 0; i < this.known_contracts.length; i++) {
					var req = self.buildGetTokensOfOwnerRequest(self.known_contracts[i])
					Promise.all([smartEcoRouter.invokeRead(req), Promise.resolve(self.known_contracts[i])])
						.then(function(values) {
							var r = values[0]
							var contract = values[1]
							var deserialized = Neon.sc.StackItem.deserialize(r["stack"][0]["value"])
							if (self.known_contracts[self.known_contracts.length-1] == contract 
							&& deserialized["value"].length == 0 && self.totalSupply == 0) {
								self.addressHasNoTokens = true
								self.$emit('isNotWaitingForDapi')

							}
							for (i=0; i< deserialized["value"].length; i++) {
								var idHex = deserialized["value"][i]["value"]["value"][0]["value"]["value"]
								console.log(idHex)
								var id = parseInt(idHex, 16)
								var uriRequest = self.buildURIRequest(id, contract)
								var ownerRequest = self.buildOwnerOfRequest(id, contract)	
								Promise.all([smartEcoRouter.invokeRead(uriRequest), smartEcoRouter.invokeRead(ownerRequest),Promise.resolve(id), Promise.resolve(contract)])
									.then(function(values) {
										console.log(values)
										self.$emit('isNotWaitingForDapi')
										var uri = self.convertHexToString(values[0]["stack"][0]["value"])
										var owner = self.convertHexToString(values[1]["stack"][0]["value"])
										self.tokens.push ({
											"token_id": values[2],
											"uri": uri,
											"owner": Neon.wallet.getAddressFromScriptHash(Neon.u.reverseHex(values[1]["stack"][0]["value"])),
											"contract": contract
										})
										self.totalSupply +=1
									})
									.catch(function(e) {
										console.log(e)
										self.$emit('isNotWaitingForDapi')
										self.unknownError = true
									})
							}
						})
						.catch(function(e) {
							console.log(e)
							self.$emit('isNotWaitingForDapi')
							self.unknownError = true
						})

				}
			},
			loadTokensForContractPage() {
				var self = this
				console.log("hello")
				self.$emit('isWaitingForDapi')
				var smartEcoRouter = new smartEco.SmartEcoRouter()
				smartEcoRouter.start()
				this.tokens = []
				var startIndex = self.current_page * self.items_per_page + 1
				var endIndex = Math.min(self.current_page * self.items_per_page + self.items_per_page + 1, self.totalSupply + 1)
	
				for (var i=startIndex; i<endIndex; i++) {
					var uriRequest = self.buildURIRequest(i, self.contract_hash)
					var ownerRequest = self.buildOwnerOfRequest(i, self.contract_hash)	

					Promise.all([smartEcoRouter.invokeRead(uriRequest), smartEcoRouter.invokeRead(ownerRequest), Promise.resolve(i)])
						.then(function(values) {
							self.$emit('isNotWaitingForDapi')
							var uri = self.convertHexToString(values[0]["stack"][0]["value"])
							var owner = self.convertHexToString(values[1]["stack"][0]["value"])
							self.tokens.push ({
								"token_id": values[2],
								"uri": uri,
								"owner": Neon.wallet.getAddressFromScriptHash(Neon.u.reverseHex(values[1]["stack"][0]["value"])),
								"contract": self.contract_hash
							})
							console.log(self.tokens)
						})
						.catch(function() {
							self.$emit('isNotWaitingForDapi')
							self.unknownError = true
						})
					}
			},
			loadAllTokensForContract() {
				var self = this
				self.$emit('isWaitingForDapi')
				var smartEcoRouter = new smartEco.SmartEcoRouter()
				smartEcoRouter.start()
				this.tokens = []
				self.totalSupply = 0
				smartEcoRouter.invokeRead(self.buildTotalSupplyRequest())
					.then(function(r) {
						console.log(r)
						console.log(parseInt(r["stack"][0]["value"], 16))
						self.totalSupply = parseInt(r["stack"][0]["value"], 16)
						console.log(self.totalSupply)
						self.total_pages = Math.ceil(self.totalSupply / self.items_per_page)
						console.log("will load for page")
						self.loadTokensForContractPage()
					})
					.catch(function() {
						self.$emit('isNotWaitingForDapi')
						self.unknownError = true
			
					})
				},
			searchForValue() {
				this.unknownError = false
				this.addressHasNoTokens = false
				if (Neon.wallet.isAddress(this.search_value)) {
					this.valid_input = true
					this.valid_text = "Succesfully validated address"
					this.address = this.search_value
					console.log(this.address)
					this.loadAllTokensForAddress()
				} else if (Neon.u.isHex(this.search_value) && this.search_value.length == 40) {
					this.valid_input = true
					this.contract_hash = this.search_value
					this.valid_text = "Succesfully validated contract"
					this.loadAllTokensForContract()
				} else {
					this.valid_input = false
				}
			},
		}, mounted: function () {
				var self = this
				var smartEcoRouter = new smartEco.SmartEcoRouter()
				smartEcoRouter.start()
				smartEcoRouter.getNetworks()
				.then(function(r) {
					console.log(r)
					if (r.networks.includes("TestNet") == false) {
						let element = self.$refs.modal.$el
            $(element).modal('show')
					}
				}).catch(function(e){
					
				})
    }
	};
</script>

<style>
 .landing-info {
	color: #000000;
	font-size: 13px;
	text-align: center;
}

.landing-info-title {
	color: #000000;
	font-size: 13px;
	font-weight: bold;
	text-align: center;
}

.success-message {
	color: #7ED321;
	font-size: 18px;
	font-weight: bold;
}

.paging-button {
	min-width: 100px;
}

</style>
