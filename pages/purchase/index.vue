<template>
  <div>
    <div class='purchase-box' :class='isBamboo ? "bamboo-purchase-box" : ""'>
      <p class='title-common'><span class='title-step' v-if='isRegister'>step1</span><span>Select A Plan</span></p>
      <p class='title-common'>{{userName}}</p>
      <PackageListComp 
        ref='packageListComp' 
        :packageList='packageList' 
        :selectedPackageId='selectedPackageId' 
        :firstPurchasePackageId='firstPurchasePackageId' 
        :showMostPopular='true'
        @onClick='choosePackage'
        @setPackageList='setPackageList'
      />

    </div>
  </div>
</template>

<script>
  import PackageListComp from '../../components/packageList.vue'
  
  export default {
    name: 'purchase',
    props: ['isBamboo'],
    components: {
      PackageListComp
    },
    data () {
      return {
        userName: localStorage.getItem('userName'),
        packageList: [],
        selectedPackageId: '',
        totalPrice: '',
        account: '',
        accountErrorMessage: '',
        password: '',
        passwordErrorMessage: '',
        email: '',
        emailErrorMessage: '',
        isRegister: this.$route.query.status === 'register',
        hasGenerate: false,
        registerId: '',
        platformId: '',
        platformName: '',
        channelId: '',
        isDisabled: false,
        invitationCode: this.$route.query.invitationCode,
        status: this.$route.query.status || '',
        webAccessToken: this.$route.query.webAccessToken || '',
        firstPurchasePackageId: null,
        showStripeModal: false,
        registrationInfo: {}
      }
    },
    mounted () {
      this.packageList = JSON.parse(localStorage.getItem('packageList') || '[]') || []
    },
    computed: {
      accesstoken () {
        const userInfo = JSON.parse(localStorage.getItem('userInfo')) || {}

        return 'Bearer' + (this.webAccessToken || userInfo.accessToken || undefined)
      },
      packageName () {
        for (let item of this.packageList) {
          if (item.id === this.selectedPackageId) {
            return item.name
          }
        }
      },
      currentPackage () {
        for (let item of this.packageList) {
          if (item.id === this.selectedPackageId) {
            return item
          }
        }
        return {}
      }
    },
    watch: {
      packageList () {
        const list = this.packageList
        
        for (let item of list) {
          if (item.firstPurchaseAwardDays > 0) {
            
            continue
          }
          if (item.isRecommended) {
            this.selectedPackageId = this.firstPurchasePackageId || item.id
            continue
          }
        }
        if (this.$route.query.packageId) {
          this.selectedPackageId = parseInt(this.$route.query.packageId)
        }
        if (this.registrationInfo.packageId) {
          this.selectedPackageId = parseInt(this.registrationInfo.packageId)
        }
        this.totalPrice = this.getTotalPrice(this.selectedPackageId)
      },
      registrationInfo () {
        this.password = this.registrationInfo.password || ''
        this.email = this.registrationInfo.email || ''
        this.platformId = this.registrationInfo.platformId || ''
        this.platformName = this.registrationInfo.platformName || ''
        this.channelId = this.registrationInfo.channelId || ''
      }
    },
    methods: {
      setPackageList (list) {
        this.packageList = list
      },
      choosePackage (event, item) {
        this.selectedPackageId = parseInt(item.id)
        this.totalPrice = this.getTotalPrice(this.selectedPackageId)
        // this.addClickAction(item)
      },
      getTotalPrice (packageId) {
        for (let i of this.packageList) {
          if (i.id === packageId) {
            return this.$refs.packageListComp.getCurrencySymbol(i.currency) + i.price
          }
        }
      },
      createNewAccount () {
        this.isRegister = true
      },
      generateRegisterId (callback) {
        doFetch({
          url: '/api/register/user-number',
          method: 'POST',
          params: {
            'deviceType': 'WEB'
          },
          success: (res) => {
            this.renderCanvas(res.data)
            localStorage.setItem('oldAccount', res.data)
            sessionStorage.setItem('oldAccount', res.data)
            callback instanceof Function && callback()
          },
          fail: (res) => {
            if (res.code === '006') {
              this.$toast(this.$t('purchase.after_24_hours'))
              if (!localStorage.getItem('account_register') && localStorage.getItem('oldAccount')) {
                localStorage.setItem('account_register', JSON.stringify({
                  id: localStorage.getItem('oldAccount'),
                  timeStamp: +new Date()
                }))
                localStorage.removeItem('oldAccount')
              }
            }
          }
        })
      },
      saveAsImage () {
        canvas2image.saveAsImage('account-canvas', '250', '60', 'png', this.$t('purchase.file_name'))
      },
      setPlatformId (id, name, channelId) {
        this.platformId = id
        this.platformName = name
        this.channelId = channelId
      },
      validateForm () {
        if (this.selectedPackageId === '') {
          this.$toast(this.$t('purchase.package_toast'))
          return false
        }
        if (this.isRegister) {
          if (!this.registerId) {
            this.$toast(this.$t('purchase.account_id_toast'))
            return false
          }
          if (this.password === '') {
            this.passwordErrorMessage = this.$t('purchase.password_empty_message')
            return false
          }
          if (this.password.length < 8 || this.password.length > 16) {
            this.passwordErrorMessage = this.$t('purchase.password_length_toast')
            return false
          }
          if (this.email && !emailVerify(this.email)) {
            this.emailErrorMessage = this.$t('purchase.email_toast')
            return false
          }
          if (this.email && !this.isEmailCorrect) {
            return false
          }
        }
        if (!this.platformId) {
          this.$toast(this.$t('purchase.platform_methods'))
          return false
        }
        return true
      },
      onSubmit () {
        if (this.needAutoGenerateRegisterId()) {
          this.isDisabled = true
          this.generateRegisterId(() => {
            this.isDisabled = false
            this.submitPurchase()
          })
        } else {
          this.submitPurchase()
        }
      },
      needAutoGenerateRegisterId () {
        if (!this.isRegister || this.registerId) {
          return false
        }
        if (this.selectedPackageId === '' || this.password === '' || this.email === '' || this.platformId === '') {
          return false
        }
        if (this.validatePassword(this.password) || !this.isEmailCorrect) {
          return false
        }
        return true
      },
      submitPurchase () {
        if (!this.isDisabled && this.validateForm()) {
          if (this.platformId === 'stripe') {
            this.showStripeModal = true
            return
          }
          this.isDisabled = true
          if (this.isRegister) {
            if (this.platformId === 'alipayGlobal' || this.platformId === 'wechatpay' || (this.platformId === 'paymentwall' && this.currentPackage.timeUnit === 'YEAR')) {
              this.registerTopup()
            } else {
              this.registerTopup('/api/register/subscription')
            }
          } else {
            if (this.platformId === 'alipayGlobal' || this.platformId === 'wechatpay' || (this.platformId === 'paymentwall' && this.currentPackage.timeUnit === 'YEAR')) {
              this.createOrder()
            } else {
              this.createSubcription()
            }
          }
          this.saveRegistrationInfo()
          this.addCheckoutAction()
        }
      },
      registerTopup (url) {
        doFetch({
          url: url || '/api/register/topup',
          headers: {
            "product-identifier": this.isBamboo ? 'bamboo' : 'panda'
          },
          method: 'POST',
          params: {
            "email": this.email || undefined,
            "number": this.registerId,
            "packageId": this.selectedPackageId,
            "password": this.password,
            "platformId": this.platformId,
            "invitationCode": this.invitationCode,
            "sourceType": "WEB",
            "webDomain": window.location.origin,
            "channelId": this.channelId || undefined,
            "source": this.getSourceType()
          },
          success: (res) => {
            if (url) {
              window.location = res.data.paymentUrl
            } else {
              this.handlePaymentSuccess(res.data)
            }
            localStorage.setItem('register_id', this.registerId)
          },
          fail: (res) => {
            if (res.code === '003') {
              this.emailErrorMessage = res.message
            }
            this.$toast(res.message)
            this.isDisabled = false
          }
        })
      },
      createOrder () {
        doFetch({
          url: '/api/orders',
          method: 'POST',
          params: {
            "account": this.account,
            "packageId": this.selectedPackageId,
            "platformId": this.platformId,
            "quantity": 1,
            "isAppending": this.status === 'appending',
            "sourceType": "WEB",
            "webDomain": window.location.origin,
            "channelId": this.channelId || undefined,
            "source": this.getSourceType()
          },
          headers: {
            'Authorization': this.accesstoken
          },
          success: (res) => {
            this.handlePaymentSuccess(res.data)
          },
          fail: (res) => {
            if (res.code === '005' && !this.isBamboo) {
              this.$router.push('/login')
            } else {
              this.$toast(res.message)
            }
            this.isDisabled = false
          }
        })
      },
      createSubcription () {
        doFetch({
          url: '/api/subscriptions',
          method: 'POST',
          params: {
            "account": this.account,
            "packageId": this.selectedPackageId,
            "paymentPlatformId": this.platformId,
            "isAppending": this.status === 'appending',
            "sourceType": "WEB",
            "webDomain": window.location.origin,
            "channelId": this.channelId || undefined,
            "source": this.getSourceType()
          },
          headers: {
            'Authorization': this.accesstoken
          },
          success: (res) => {
            window.location = res.data.paymentUrl
          },
          fail: (res) => {
            if (res.code === '005' && !this.isBamboo) {
              this.$router.push('/login')
            } else {
              this.$toast(res.message)
            }
            this.isDisabled = false
          }
        })
      },
      stripeCreateCustomer (stripeToken, stripeName) {
        doFetch({
          url: '/api/stripe-create-customer',
          params: {
            "number": this.isRegister ? this.registerId : this.account,
            "token": stripeToken
          },
          method: 'POST',
          success: (res) => {
            if (this.isRegister) {
              this.saveRegistrationInfo({stripeName})
              this.stripeRegisterSubscribe()
            } else {
              this.stripeSubscribe()
            }
          },
          fail: (res) => {
            if (res.code === '001') {
              this.$refs.stripeModalComp.addCardErrorStyle()
            } else {
              this.$toast(res.message)
            }
            this.$refs.stripeModalComp.isDisabled = false
          }
        })
      },
      stripeRegisterSubscribe () {
        doFetch({
          url: '/api/register/subscription',
          headers: {
            "product-identifier": this.isBamboo ? 'bamboo' : 'panda'
          },
          method: 'POST',
          params: {
            "email": this.email || undefined,
            "number": this.registerId,
            "packageId": this.selectedPackageId,
            "password": this.password,
            "platformId": this.platformId,
            "invitationCode": this.invitationCode,
            "sourceType": "WEB",
            "webDomain": window.location.origin,
            "channelId": this.channelId || undefined,
            "source": this.getSourceType()
          },
          success: (res) => {
            this.$router.push({path: this.isBamboo ? '/verify-order-status' : '/verify-paymentwall', query: {orderNumber: res.data.orderNumber, seconds: 5}})
            localStorage.setItem('register_id', this.registerId)
          },
          fail: (res) => {
            if (res.code === '003') {
              this.showStripeModal = false
              this.emailErrorMessage = res.message
            } else if (res.code === '001') {
              this.$refs.stripeModalComp.addCardErrorStyle()
            } else {
              this.$toast(res.message)
            }
            this.isDisabled = false
            this.$refs.stripeModalComp.isDisabled = false
          }
        })
      },
      stripeSubscribe () {
        doFetch({
          url: '/api/subscriptions',
          method: 'POST',
          params: {
            "account": this.account,
            "packageId": this.selectedPackageId,
            "paymentPlatformId": this.platformId,
            "isAppending": this.status === 'appending',
            "sourceType": "WEB",
            "webDomain": window.location.origin,
            "channelId": this.channelId || undefined,
            "source": this.getSourceType()
          },
          headers: {
            'Authorization': this.accesstoken
          },
          success: (res) => {
            this.$router.push({path: this.isBamboo ? '/verify-order-status' : '/verify-paymentwall', query: {orderNumber: res.data.orderNumber, seconds: 5}})
          },
          fail: (res) => {
            if (res.code === '005' && !this.isBamboo) {
              this.$router.push('/login')
            } else if (res.code === '001') {
              this.$refs.stripeModalComp.addCardErrorStyle()
            } else {
              this.$toast(res.message)
            }
            this.isDisabled = false
            this.$refs.stripeModalComp.isDisabled = false
          }
        })
      },
      saveRegistrationInfo (otherInfo) {
        localStorage.setItem('registrationInfo', JSON.stringify({
          packageId: this.selectedPackageId,
          password: this.password,
          email: this.email,
          platformId: this.platformId,
          platformName: this.platformName,
          channelId: this.channelId,
          ...otherInfo
        }))
      },
      handlePaymentSuccess (orderInfo) {
        if (orderInfo.platformId === 'wechatpay') {
          this.$router.push({path: '/wechat-pay', query: {orderNumber: orderInfo.orderNumber}})
          localStorage.setItem('code_url', orderInfo.detail.code_url)
        } else {
          window.location.href = orderInfo.detail.redirect_url
        }
      },
      renderCanvas (registerId) {
        const canvas = document.getElementById('account-canvas')
        const context = canvas.getContext('2d')

        this.hasGenerate = true
        context.fillStyle = '#47B96D'
        context.font = 'bold 45px Arial'
        context.fillText(registerId, 0, 40)
        this.registerId = registerId
      },
      addClickAction (item) {
        this.$ga.set({'currencyCode': item.currency})
        this.$ga.ecommerce.addProduct({
          'id': item.id,
          'name': item.name,
          'price': item.price,
          'list': 'Panda VPN',
          'currency': item.currency
        })
        this.$ga.ecommerce.setAction('click', {'list': 'Panda VPN'})
        window.ga && window.ga('send', 'event', 'UX', 'click', 'Results')
      },
      addCheckoutAction () {
        const item = this.currentPackage

        this.$ga.ecommerce.addProduct({
          'id': item.id,
          'name': item.name,
          'price': item.price,
          'list': 'Panda VPN',
          'currency': item.currency
        })
        this.$ga.ecommerce.setAction('checkout', {'step': 1, 'options': this.platformId})
        window.ga && window.ga('send','pageview')
        sessionStorage.setItem('checkoutInfo', JSON.stringify({...item, 'platformId': this.platformId}))
      },
      handleInvitationCode () {
        const code = sessionStorage.getItem('invitationCode')

        if (this.$route.query.invitationCode) {
          sessionStorage.setItem('invitationCode', this.invitationCode)
        } else if (this.isRegister && code) {
          this.$router.replace(`/purchase?status=register&invitationCode=${code}`)
        }
      },
      validatePassword (password) {
        if (password && (password.length < 8 || password.length > 16)) {
          return this.$t('purchase.password_length_toast')
        }
        return ''
      },
      validateEmail (email) {
        if (email && (email.length < 6 || email.length > 30 || !emailVerify(email))) {
          this.isEmailCorrect = false
          return this.$t('purchase.email_toast')
        }
        return ''
      },
      verifyEmailByAPI (e) {
        doFetch({
          url: '/api/register/validation/email',
          method: 'POST',
          params: {
            email: e.target.value
          },
          async: false,
          success: () => {
            this.isEmailCorrect = true
          },
          fail: (res) => {
            if (res.code === '003') {
              this.emailErrorMessage = res.message
            } else {
              this.emailErrorMessage = this.$t('purchase.email_toast')
            }
            this.isEmailCorrect = false
          }
        })
      },
      getSourceType () {
        if (this.$route.query.from) {
          return this.$route.query.from
        } else if (!isMobile()) {
          return 'web'
        } else if (isAndroid()) {
          return 'h5-android'
        } else if (isIOS()) {
          return 'h5-ios'
        }
        return ''
      },
      closeStripeModal () {
        this.showStripeModal = false
      }
    }
  }
</script>

<style lang='scss' scoped>
  .title-common {
    font-size: 20px;
    font-weight: 500;
    color: #333;
    text-align: center;
  }
</style>
