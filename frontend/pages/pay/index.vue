<template>
	<view class="app">
		<view class="price-box">
			<text>支付金额</text>
			<text class="price">{{orderInfo.order_price}}</text>
		</view>
		<view class="count-down-box" v-if="left_time">
			<text>请在 </text>
			<view class="time">
				<u-count-down @finish="reGetData" :time="left_time"></u-count-down>
			</view>
			<text> 内完成支付</text>
		</view>

		<view class="pay-type-list">
			<view class="type-item b-b" @click="changePayType('wechat')" v-if="payTypeArr.indexOf('wechat') > -1">
				<text class="icon iconfont icon19weixinzhifu"></text>
				<view class="con">
					<text class="tit">微信支付</text>
				</view>
				<label class="radio">
					<radio value="" color="#fa436a" :checked='payType == "wechat"' />
					</radio>
				</label>
			</view>
			<view class="type-item b-b" @click="changePayType('alipay')" v-if="payTypeArr.indexOf('alipay') > -1">
				<text class="icon iconfont iconalipay"></text>
				<view class="con">
					<text class="tit">支付宝支付</text>
				</view>
				<label class="radio">
					<radio value="" color="#fa436a" :checked='payType == "alipay"' />
					</radio>
				</label>
			</view>
		</view>
		<text class="mix-btn" @click="confirm">确认支付</text>
	</view>
</template>

<script>
	import {
		mapState
	} from 'vuex';
	import Account from '@/common/Account';
	var timer;
	export default {
		components: {
			
		},
		data() {
			return {
				checkAuth: true,
				payType: "wechat",
				orderInfo: {},
				// #ifdef H5
				openId: null,
				// #endif
				showTransfer: false
			};
		},
		computed: {
			...mapState({
				appInfo: state => state.appInfo
			}),
			left_time() {
				let t = this.appInfo?.config?.shop?.order?.order_life || 0;
				if (t == 0) return 0;
				let left_time = this.orderInfo.created_at + t * 60 - Math.ceil(Date.now() / 1000);
				return parseInt(left_time) * 1000;
			},
			payTypeArr() {
				let res = [];
				// #ifdef H5
				// res = ['wechat', 'alipay'];
				if (this.$wxApi.isweixin()) {
					res = ['wechat']
				} else res = ['wechat', 'alipay']
				// #endif
				// #ifdef MP-WEIXIN
				res = ['wechat']
				// #endif
				// #ifdef APP-PLUS
				res = ['wechat', 'alipay'];
				// #endif
				// #ifdef MP-ALIPAY | MP-TOUTIAO
				res = ['alipay']
				// #endif
				return res
			}
		},
		async onReady() {
			// #ifdef H5
			if (this.$wxApi.isweixin()) {
				Account.init('WechatH5').login2()
			}
			// #endif
		},
		async onLoad(options) {
			options = await this.onLoadStart(options);
			this.order_sn = options.order_sn
			this.getDefaultPayType()
			this.getData()

			this.$on('onPaySuccess', (e, res) => {
				this.paySuccess()
			})
			this.$on('onPayFail', (e, res) => {
				uni.showToast({
					title: e,
					'icon': 'none'
				})
			});
			timer = setInterval(() => {
				this.getData()
			}, 3000);
		},
		onUnload() {
			clearInterval(timer)
		},
		methods: {
			getDefaultPayType() {
				this.payType = this.payTypeArr[0] || '';
			},
			async getData() {
				const query = `
					query($order_sn: String!) {
						order(order_sn: $order_sn) {
							is_pay
							status
							order_price
							created_at
						}
					}
				`
				const result = await this.$gql.fetch(query, { order_sn: this.order_sn });
				const err = result.getError('order');
				if (err) return result.show(err);
				const { order } = result.get();
				this.orderInfo = order;
				this.checkOrder()
			},
			async handleTransferOk() {
				this.navTo()
			},
			//选择支付方式
			changePayType(type) {
				this.payType = type;
			},
			confirm() {
				if (this.payTypeArr.length == 1) this.payType = this.payTypeArr[0]
				// #ifdef H5
				if (this.$wxApi.isweixin()) {
					this.confirmH5WEIXIN()
				} else {
					this.confirmWap()
				}
				// #endif
				// #ifdef MP-WEIXIN
				this.confirmMPWEIXIN()
				// #endif
				// #ifdef APP-PLUS
				this.confirmApp()
				// #endif
				// #ifdef MP-ALIPAY
				this.confirmMPALIPAY()
				// #endif
				// #ifdef MP-TOUTIAO
				this.confirmMPTOUTIAO()
				// #endif

			},
			// #ifdef H5
			//确认支付
			confirmH5WEIXIN() {
				let pay_method = this.$wxApi.isweixin() ? 'mp' : 'wap';
				let form = {
					out_order_sn: this.order_sn,
					pay_type: this.payType,
					scene: 'shop',
					pay_method,
					appid: uni.getStorageSync('appId'),
					openid: uni.getStorageSync('openId')
				}
				uni.showLoading()
				this.$http.post('shop/api/order/pay', form).then(data => {
					uni.hideLoading()
					if (data) {
						let wxpay = typeof data == 'string' ? JSON.parse(data) : data
						if (typeof WeixinJSBridge == "undefined") {
							if (document.addEventListener) {
								document.addEventListener('WeixinJSBridgeReady', this.onBridgeReady(wxpay),
									false);
							} else if (document.attachEvent) {
								document.attachEvent('WeixinJSBridgeReady', this.onBridgeReady(wxpay));
								document.attachEvent('onWeixinJSBridgeReady', this.onBridgeReady(wxpay));
							}
						} else {
							return this.onBridgeReady(wxpay);
						}
					}

				}).catch(e => {
					uni.hideLoading()
				})
			},
			onBridgeReady(wxpay) {
				WeixinJSBridge.invoke('getBrandWCPayRequest', {
					appId: wxpay.appId,
					timeStamp: wxpay.timeStamp,
					nonceStr: wxpay.nonceStr,
					package: wxpay.package,
					signType: wxpay.signType,
					paySign: wxpay.paySign
				}, res => {
					if (res.err_msg == "get_brand_wcpay_request:ok") { //支付成功
						this.$emit('onPaySuccess')
						return true;
					} else { //支付失败
						this.$emit('onPayFail', '支付失败')
					}
					return false;
				})
			},
			confirmWap() {
				let pay_method = 'wap'
				let form = {
					out_order_sn: this.order_sn,
					pay_type: this.payType,
					scene: 'shop',
					pay_method
				}
				uni.showLoading()
				this.$http.post('shop/api/order/pay', form).then(res => {
					uni.hideLoading()
					if (this.payType == 'alipay') {
						var el = document.createElement("div");
						el.setAttribute('id', 'alipaysubmit-box')
						el.innerHTML = res;
						document.body.appendChild(el);
						let form = document.forms['alipay_submit']
						let data = this.$tools.serializeForm('alipay_submit')
						let url = form.action
						document.body.removeChild(el)
						window.location.href = url + '&' + data						
					} else if (this.payType == 'wechat') {
						window.location.href = res.h5_url;
					}
				})
			},
			// #endif

			// #ifdef MP-WEIXIN
			confirmMPWEIXIN() {
				let pay_method = 'miniapp'
				let form = {
					out_order_sn: this.order_sn,
					pay_type: this.payType,
					scene: 'shop',
					pay_method,
					appid: uni.getStorageSync('appId'),
					openid: uni.getStorageSync('openId')
				}
				uni.showLoading()
				this.$http.post('shop/api/order/pay', form).then(data => {
					uni.hideLoading()
					if (data) {
						let wxpay = typeof data == 'string' ? JSON.parse(data) : data
						uni.requestPayment({
							provider: 'wxpay',
							timeStamp: wxpay.timeStamp,
							nonceStr: wxpay.nonceStr,
							package: wxpay.package,
							signType: wxpay.signType,
							paySign: wxpay.paySign,
							success: res => {
								this.$emit('onPaySuccess')
							},
							fail: () => {
								this.$emit('onPayFail')
							},
							complete: () => {}
						});
					}
				}).catch(e => {
					uni.hideLoading()
				})
			},
			// #endif

			// #ifdef APP-PLUS
			confirmApp() {
				let pay_method = 'app'
				let form = {
					out_order_sn: this.order_sn,
					pay_type: this.payType,
					scene: 'shop',
					pay_method,
					appId: uni.getStorageSync('appId'),
					openId: uni.getStorageSync('openId')
				}
				uni.showLoading()
				this.$http.post('shop/api/order/pay', form).then(data => {
					uni.hideLoading()
					if (data) {
						uni.requestPayment({
							provider: this.payType == 'wechat' ? 'wxpay' : this.payType,
							orderInfo: data,
							success: res => {
								if (res.errMsg == 'requestPayment:ok') {
									this.$emit('onPaySuccess')
								} else {
									this.$emit('onPayFail', '支付失败')
								}
							},
							fail: res => {
								alert(JSON.stringify(res))
								this.$emit('onPayFail', '支付失败')
							}
						})
					}
				}).catch(e => {
					uni.hideLoading()
				})
			},
			// #endif

			// #ifdef MP-ALIPAY
			confirmMPALIPAY() {
				let pay_method = 'mp'
				let form = {
					out_order_sn: this.order_sn,
					pay_type: this.payType,
					scene: 'shop',
					pay_method,
					appId: uni.getStorageSync('appId'),
					openId: uni.getStorageSync('openId')
				}
				uni.showLoading()
				this.$http.post('shop/api/order/pay', form).then(data => {
					uni.hideLoading()
					if (data) {
						uni.requestPayment({
							provider: this.payType,
							orderInfo: data,
							success: res => {
								if (res.resultCode == 9000) {
									this.$emit('onPaySuccess')
								} else {
									this.$emit('onPayFail', '支付失败')
								}
							},
							fail: res => {
								this.$emit('onPayFail', '支付失败')
							}
						})
					}
				}).catch(e => {
					uni.hideLoading()
				})
			},
			// #endif
			paySuccess() {
				uni.redirectTo({
					url: '/pages/pay/success'
				})
			},
			navTo(title = '', url = '/pages/order/index?state=0') {
				if (title) uni.showToast({
					title,
					icon: 'none'
				})
				setTimeout(() => {
					uni.hideToast()
					uni.redirectTo({
						url
					})
				}, 500)
			},
			checkOrder() {
				if (this.orderInfo.status == -1) {
					if (timer) {
						clearInterval(timer);
						timer = null;
					}
					return this.navTo('订单已失效')
				} else if (this.orderInfo.is_pay == 1) {
					if (timer) {
						clearInterval(timer);
						timer = null;
					}
					return this.navTo("订单已支付")
				}
			},
			reGetData() {
				setTimeout(() => {
					this.getData()
				}, 1500)
			}
		}
	}
</script>

<style lang='scss'>
	.app {
		width: 100%;
	}

	.price-box {
		background-color: #fff;
		height: 265upx;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		font-size: 28upx;
		color: #909399;

		.price {
			font-size: 50upx;
			color: #303133;
			margin-top: 12upx;

			&:before {
				content: '￥';
				font-size: 40upx;
			}
		}
	}

	.pay-type-list {
		margin-top: 20upx;
		background-color: #fff;
		padding-left: 60upx;

		.type-item {
			height: 120upx;
			padding: 20upx 0;
			display: flex;
			flex-direction: row;
			justify-content: space-between;
			align-items: center;
			padding-right: 60upx;
			font-size: 30upx;
			position: relative;
		}

		.icon {
			width: 100upx;
			font-size: 52upx;
		}

		.icon-erjiye-yucunkuan {
			color: #fe8e2e;
		}

		.icon19weixinzhifu {
			color: #36cb59;
		}

		.iconalipay {
			color: #01aaef;
		}

		.tit {
			font-size: $font-lg;
			color: $font-color-dark;
			margin-bottom: 4upx;
		}

		.con {
			flex: 1;
			display: flex;
			flex-direction: column;
			font-size: $font-sm;
			color: $font-color-light;
		}
	}

	.mix-btn {
		display: flex;
		align-items: center;
		justify-content: center;
		width: 630upx;
		height: 80upx;
		margin: 80upx auto 30upx;
		font-size: $font-lg;
		color: #fff;
		background-color: $base-color;
		border-radius: 10upx;
		box-shadow: 1px 2px 5px rgba(219, 63, 96, 0.4);
	}

	.count-down-box {
		display: flex;
		flex-direction: row;
		justify-content: center;
		margin-top: 20rpx;
		padding: 20upx 0;
		font-size: $font-base;

		.time {
			color: $base-color;
		}

	}
</style>
