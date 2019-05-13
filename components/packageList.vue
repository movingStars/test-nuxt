<template>
  <div>
    <div class='package-list-pc'>
      <div 
        v-for='(item, index) in packageList' 
        :class='["info-item", item.id === selectedPackageId ? "selected" : "", item.firstPurchaseAwardDays > 0 ? " first-purchase" : ""]'
        :key='index'
        @click='$emit("onClick", $event, item)'
      >
        <div>
          <img :src='getImageByPackageType(item)' alt=''>
          <div>
            <div>
              <div class='name' v-if='item.firstPurchaseAwardDays > 0'>
                <span>首充</span>
                <span>{{item.name}}</span>
                <span>{{getAwardTime(item.firstPurchaseAwardDays)}}</span>
              </div>
              <div class='name' v-else>{{item.name}}</div>
              <p class='price'><span>{{getCurrencySymbol(item.currency)}}</span>{{item.unitPrice}}</p>
              <div class='time'>每月</div>
            </div>
            <div class='billed'>花费</div>
          </div>
          <div v-if='showMostPopular && item.isRecommended && item.firstPurchaseAwardDays <= 0 && !firstPurchasePackageId' class='most-popular'>most popullar</div>
          <div class='off-box' v-if='item.price !== item.listPrice'>
            <div>
              <span>节省</span>
              <p>{{item.discount}}</p>
            </div>
          </div>
          <div v-if='item.firstPurchaseAwardDays > 0' class='count-down'>
            <ul>
              <li>{{countDown.hour.split('')[0]}}</li><li>{{countDown.hour.split('')[1]}}</li><li class='colon'>:</li>
              <li>{{countDown.minute.split('')[0]}}</li><li>{{countDown.minute.split('')[1]}}</li><li class='colon'>:</li>
              <li>{{countDown.second.split('')[0]}}</li><li>{{countDown.second.split('')[1]}}</li>
            </ul>
          </div>
          <router-link v-if='showPurchaseButton' class='subscribe' tag='button' :to='"/purchase?packageId="+item.id'>订阅</router-link>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
  import {mapState} from 'vuex'

  export default {
    props: ['packageList', 'selectedPackageId', 'firstPurchasePackageId', 'showPurchaseButton', 'showMostPopular'],
    data () {
      return {
        countDown: {hour: '', minute: '', second: ''},
        intervalId: null
      }
    },
    computed: {
      ...mapState({
        currentLang: 'currentLang'
      })
    },
    mounted () {
      if (!localStorage.getItem('timeOfFirstEnter') && !sessionStorage.getItem('timeOfFirstEnter')) {
          localStorage.setItem('timeOfFirstEnter', +new Date())
      }
      this.getPackageList()
    },
    destroyed () {
      this.intervalId = null
    },
    methods: {
      getPackageList () {
        const userInfo = JSON.parse(localStorage.getItem('userInfo')) || {}

        fetch("https://api.myjson.com/bins/1gqr02").then((res) => {
          return res.json()
        }).then((res) => {
          debugger
          this.$emit('setPackageList', res.data || [])
        })
      },
      getImageByPackageType (item) {
      },
      getCurrencySymbol (currency) {
        switch (currency) {
          case 'CNY':
            return '￥'
          case 'USD':
            return '$'
        }
      },
      getAwardTime (awardDays) {
        const year = Math.floor(awardDays / 365)
        const month = Math.floor((awardDays - year * 365) / 30)
        let yearText = '', monthText = ''

        return yearText || monthText
      },
      getTimeUnit (item) {
        switch (item.timeUnit.toLowerCase()) {
          case 'year':
            return '年'
          case 'month':
            return item.timeQuantity > 1 && (!this.currentLang || this.currentLang === 'en-US') ? 'months' : 'month'
        }
      },
      packageListSort () {
        const list = [...this.packageList]
        const timeUnitChart = {
          'DAY': 1,
          'WEEK': 2,
          'MONTH': 3,
          'YEAR': 4
        }
        
        return list.sort((item1, item2) => {
          const x = timeUnitChart[item1.timeUnit] - timeUnitChart[item2.timeUnit]
          const y = item1.timeQuantity - item2.timeQuantity
          const z = item1.firstPurchaseAwardDays - item2.firstPurchaseAwardDays

          return x == 0 ? y == 0 ? z : y : x
        })
      },
      startCountingDown (startFunc = ()=>{}, endFunc = ()=>{}) {
        const startTime = localStorage.getItem('timeOfFirstEnter')
        const userInfo = JSON.parse(localStorage.getItem('userInfo')) || {}
        const countdownTime = countDownMinutes * 60 * 1000
        const nowTime = +new Date()
        let times = (parseInt(startTime) + countdownTime - nowTime) / 1000

        if (userInfo.role !== 'NORMAL') {

          const countDownFunc = () => {
            let hour=0,
                minute=0,
                second=0

            if (times > 0) {
              hour = Math.floor(times / (60 * 60)) + ''
              minute = Math.floor(times / 60) - (hour * 60) + ''
              second = Math.floor(times) - (hour * 60 * 60) - (minute * 60) + ''
            }
            if (times <= 0) {
              clearInterval(this.intervalId)
              localStorage.removeItem('timeOfFirstEnter')
              sessionStorage.setItem('timeOfFirstEnter', startTime)
            }
            if (!startTime) {
              clearInterval(this.intervalId)
            }
            if (hour <= 9) hour = '0' + hour
            if (minute <= 9) minute = '0' + minute
            if (second <= 9) second = '0' + second

            this.countDown = {
              hour,
              minute,
              second
            }
            times--
          }
          countDownFunc()
          startFunc()
          this.intervalId = setInterval(() => {
            countDownFunc()
          },1000)
        }
        if (times <= 0) {
          endFunc()
          clearInterval(this.intervalId)
        }
      }
    }
  }
</script>

<style lang='scss' scoped>
  .package-list-pc {
    width: 100%;
    height: 340px;
    margin: 100px auto 0;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-wrap: wrap;
    .info-item {
      width: 22.222%;
      height: 100%;
      text-align: center;
      border: 1px solid #E8EBF5;
      border-radius: 4px;
      background: transparent;
      position: relative;
      z-index: 1;
      letter-spacing: .5px;
      cursor: pointer;
      >div {
        border: 3px solid transparent;
      }
      img {
        height: 120px;
        margin: -33% auto 0;
        z-index: 2;
      }
      .name {
        font-size: 18px;
        line-height: 80px;
        font-weight: bold;
        margin-top: 10px;
        min-height: 80px;
      }
      .price {
        font-size: 46px;
        color: #47B96D;
        margin: 4% auto 3%;
        font-weight: bold;
        span {
          display: inline-block;
          font-size: #333;
          position: relative;
          top: -15px;
          left: -2px;
        }
      }
      .time {
        font-size: 16px;
        color: #999999;
        margin-bottom: 15px;
      }
      .billed {
        font-size: 12px;
        color: #999999;
        line-height: 18px;
        opacity: .75;
      }
      .billed span {
        color: #F6735C;
        text-decoration: line-through;
      }
      .off-box {
        width: 72px;
        height: 72px;
        position: absolute;
        top: 0;
        right: 0;
        transform: translateX(25%) translateY(-25%);
        font-size: 16px;
        color: #fff;
        vertical-align: middle;
        text-align: center;
        div {
          position: absolute;
          width: 100%;
          top: 50%;
          left: 0;
          transform: translateY(-50%);
          span {
            display: inline-block;
            width: 80%;
            overflow-wrap: break-word;
          }
          p{
            font-size: 20px;
          }
        }
      }
      .most-popular {
        display: inline-block;
        font-size: 14px;
        color: #fff;
        background: #47B96D;
        padding: 0 20px;
        margin: 25px auto 0;
        height: 30px;
        line-height: 30px;
        border-radius: 15px;
      }
      &::before {
        display: inline-block;
        content: '';
        width: 20px;
        height: 20px;
        background: #fff;
        border: 1px solid #E8EBF5;
        border-radius: 50%;
        position: absolute;
        right: 10px;
        bottom: 10px;
      }
    }
    .first-purchase {
      border: 1px solid #fff;
      >div {
        height: 100%;
        border: 3px solid #F6735C;
        box-sizing: border-box;
        border-radius: 4px;
      }
      .name {
        span {
          color: #0e0e0e;
          display: block;
          &:nth-child(1) {
            font-size: 12px;
            font-weight: 500;
            line-height: 20px;
          }
          &:nth-child(2) {
            font-size: 18px;
            line-height: 30px;
          }
          &:nth-child(3) {
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 12px;
            line-height: 16px;
            min-height: 26px;
            color: #F5654B;
            width: 80%;
            margin: 6px auto 0;
          }
        }
      }
      .price {
        color: #F6735C;
        margin: 4% auto 3%;
      }
      .time {
        margin-bottom: 15px;
      }
      .count-down {
        display: inline-block;
        margin: 20px auto 0;
        div {
          line-height: 19px;
          text-align: left;
          color: #F6735C;
        }
        ul {
          li {
            float: left;
            width: 20px;
            height: 26px;
            line-height: 26px;
            color: #fff;
            font-size: 20px;
            text-align: center;
            background: rgba($color: #000000, $alpha: .5);
            border-radius: 1px;
            &:not(:last-child) {
              margin-right: 1px;
            }
            &.day, &.colon {
              background: transparent;
              font-size: 12px;
              color: #434343;
            }
            &.day {
              width: auto;
              padding: 0 3px;
            }
            &.colon {
              width: auto;
              margin: 0 1px;
            }
          }
        }
      }
    }
    .info-item:not(.selected):hover {
      border: 1px solid #47B96D;
    }
    .info-item.selected {
      background: linear-gradient(-31deg, #2FAD66 0%, #9DE686 100%);
      border: 1px solid #fff;
      .name, .price, .time, .billed, .billed span {
        color: #fff;
      }
      .subscribe {
        border: none;
        cursor: pointer;
      }
      .most-popular {
        background: #fff;
        color: #47B96D;
      }
      &::before {
        border: 1px solid #fff;
      }
      &::after {
        display: inline-block;
        content: '';
        width: 10px;
        height: 5px;
        border-left: 2px solid #47B96D;
        border-bottom: 2px solid #47B96D;
        transform: rotateZ(-45deg);
        position: absolute;
        right: 15px;
        bottom: 19px;
      }
    }
    .first-purchase.selected {
      background-image: linear-gradient(31deg, #F45438 0%, #F6765F 100%);
      >div {
        border: 3px solid transparent;
      }
      .name {
        span {
          color: #fff;
          &:nth-child(3) {
            color: #F5654B;
          }
        }
      }
      .price {
        color: #fff;
      }
      .count-down {
        div {
          color: #fff;
        }
        ul {
          li {
            &.day, &.colon {
              color: #fff;
            }
          }
        }
      }
      &::after {
        border-left: 2px solid #F6735C;
        border-bottom: 2px solid #F6735C;
      }
    }
    .info-item:not(:last-child) {
      margin-right: 3%;
    }
  }
  .package-list-h5 {
    display: none;
  }
</style>
