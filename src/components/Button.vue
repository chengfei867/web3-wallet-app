<script setup>
import { ref } from 'vue'
import { showNotify,showDialog } from 'vant'
import * as bip39 from 'bip39'
import 'vant/es/dialog/style'
import "vant/es/notify/style"
import "vant/es/picker/style"
import {hdkey} from "ethereumjs-wallet"
import store from "store2";
import Web3 from 'web3'
import Tx from 'ethereumjs-tx'

//用于控制PasswordDialog显示
const showPasswordDialog = ref(false)
//控制是否显示助记词文本域
const showMn = ref(false)
//用于控制ConfirmMnemonicDialog显示
const showConfirmMnemonic = ref(false)
//用户创建钱包时输入的密码
const password = ref('')
//用于存储生成的助记词
const mnemonic = ref('')
//用于存储用户输入的助记词
const inputMnemonic = ref('')
//用于控制用户账号界面
const showAccount = ref(false)
//用户账号列表
const accounts = ref([])
//转账金额
const money = ref('')
//当前选中的账户
const from = ref('')
//转帐方私钥
const privateKey = ref('')
//收款方地址
const to = ref('')
//控制选择框
const showPicker = ref(false)
//账户显示框
const fieldValue = ref('')
//选择框展示的数据格式
const columns = ref([])

//连接区块链
const web3 = new Web3(Web3.givenProvider || 'localhost:7545');

//点击创建钱包按钮弹出提示框要求用户输入密码
  const createWallet=async ()=>{
    // console.log('createWallet函数')
    showPasswordDialog.value=true
  }

  //点击导入钱包按钮
  const importWallet=async ()=>{

  }

  //用户确认提交密码
  const confirmPassword = ()=>{
    //输入的密码为空，提示
    if (password.value===''){
      showNotify({ type: 'danger', message: '请输入密码!' })
      //本地没有存储密码，表示首次创建账户
    }else if(!store('password')){
      //1 通过bip39生成助记词
      mnemonic.value = bip39.generateMnemonic()
      //进入账号创建步骤
      showMn.value=true
    }else if(password.value!==store('password')){
      showNotify({ type: 'danger', message: '密码错误!' })
    }else{
      //展示现有账号
      accounts.value=store('accounts')
      columns.value = accounts.value.map(({ address:text, balance:value,privateKey}) => ({ text, value ,privateKey}))
      showAccount.value=true
    }
  }

  //用户取消提交密码
  const cancelPassword = ()=>{
    password.value=''
  }

  //用户点击确认保存助记词按钮
  const saveMnemonic=()=>{
    showConfirmMnemonic.value=true
  }

//用户确认提交助记词
const confirmMnemonic = async ()=>{
  showMn.value=false
  //如果输入的助记词与生成的助记词不相同
  if (inputMnemonic.value!==mnemonic.value){
    password.value=store('password')
    mnemonic.value=store('mnemonic')
    showNotify({ type: 'danger', message: '输入的助记词与生成的助记词不一致' })
  }else{
    //创建钱包
    //2 根据助记词创建钱包
    const seed = await bip39.mnemonicToSeed(mnemonic.value)
    const hdWallet = hdkey.fromMasterSeed(seed)
    //3 根据钱包生成密钥对
    const keypair = hdWallet.derivePath(`m/44'/60'/0'/0/0`)
    const wallet = keypair.getWallet()
    //4 获取账户信息
    const lowerCaseAddress = wallet.getAddressString()
    const publicKey= wallet.getPublicKey()
    const checkSumAddress = wallet.getChecksumAddressString()
    const privateKey = wallet.getPrivateKey().toString('hex')
    //5 生成keyStore
    const keyStore =await wallet.toV3(password.value)
    const accountInfo = {
      address:lowerCaseAddress,
      publicKey,
      privateKey,
      keyStore,
      mnemonic:mnemonic.value,
      balance:0
    }
    //将第一次创建的钱包信息保存在本地
    store('password',password.value)
    store('mnemonic',mnemonic.value)
    store(`accounts`,[accountInfo])
    accounts.value=[accountInfo]
    showAccount.value=true
  }
}

//用户取消提交助记词
const cancelMnemonic = ()=>{
  inputMnemonic.value=''
}

//选择账户
const selectAccount = ()=>{

}

//选择框确认
const onConfirm = ({ selectedOptions }) => {
  from.value = selectedOptions[0].text
  privateKey.value = selectedOptions[0].privateKey
  showPicker.value = false;
  fieldValue.value = selectedOptions[0].text;
};

//转账
const send = async ()=>{
  //1 构建转账信息
  //使用该账户转账次数作为随机数
  const nonce =await web3.eth.getTransactionCount(from.value)
  //获取预计转账的gas费
  const gasPrice =await web3.eth.getGasPrice()
  //将转账金额转换成wei
  const value = web3.utils.toWei(money.value)
  //构建转账结构
  const rawTx = {
    from:from.value,
    to:to.value,
    nonce,
    gasPrice,
    value,
    data:"0x00000"
  }
  //2 对交易签名
  //gas估算
  rawTx.gas=await web3.eth.estimateGas(rawTx)
  //实现私钥签名
  const tx = new Tx(rawTx)
  const sk = new Buffer(privateKey.value,'hex')
  tx.sign(sk)
  //生成serializedTx
  const serializedTx = `0x${tx.serialize().toString('hex')}`
  //3 开始转账
  const trans = web3.eth.sendSignedTransaction(serializedTx)
  trans.on('transactionHash',(txId)=>{
    console.log('交易ID：',txId)
  })
  trans.on('receipt',(res)=>{
    console.log('节点确认：',res)
    //获取收款方余额
    web3.eth.getBalance(from.value).then((res)=>{
      console.log('发送方余额',web3.utils.fromWei(res,'ether'))
    })
    web3.eth.getBalance(to.value).then((res)=>{
      console.log('收款方余额'+web3.utils.fromWei(res,'ether'))
    })
  })

}


</script>

<template>
  <van-space>
    <van-button type="primary" @click="createWallet">创建钱包</van-button>
    <van-button type="success" @click="importWallet">导入钱包</van-button>
    <van-dialog
        v-model:show="showPasswordDialog"
        title="请输入密码"
        show-cancel-button
        @confirm="confirmPassword"
        @cancel="cancelPassword">
        <van-field
          v-model="password"
          type="password"
          name="密码"
          label="密码"
          placeholder="密码">
        </van-field>
    </van-dialog>
  </van-space>
  <van-space>
    <div v-show="showMn">
      <van-field type='textarea' label="助记词" v-model="mnemonic"/>
      <van-button size="mini" @click="saveMnemonic">我已保存助记词</van-button>
    </div>
    <div v-show="showAccount">
      <van-field
          v-model="fieldValue"
          is-link
          readonly
          label="账户"
          placeholder="选择账户"
          @click="showPicker = true"
      />
      <van-popup v-model:show="showPicker" round position="bottom">
        <van-picker
            :columns="columns"
            @cancel="showPicker = false"
            @confirm="onConfirm"
        />
      </van-popup>
      <van-field type='text' label="收款地址" v-model="to" placeholder="请输入收款方地址"/>
      <van-field type='text' label="转账金额" v-model="money" placeholder="请输入转账金额"/>
      <van-button type="primary" @click="send">转账</van-button>
    </div>
    <van-dialog
        v-model:show="showConfirmMnemonic"
        title="请输入助记词"
        show-cancel-button
        @confirm="confirmMnemonic"
        @cancel="cancelMnemonic">
      <van-field
          v-model="inputMnemonic"
          type="text"
          name="助记词"
          label="助记词"
          placeholder="请输入助记词">
      </van-field>
    </van-dialog>
  </van-space>

</template>

<style scoped lang="less">

</style>