<script>
	import {
		mapActions,
		mapMutations,
		mapState
	} from 'vuex'
	import Account from '@/common/Account'
	import config from '@/common/config';
	export default {
		onLaunch: async function() {
			this.$msg.init(config.message.key);
			await this.getAppInfo()
			await this.login()
			await this.getUserinfo().catch(e => {});
			if (config.message.enable) this.subscribe()
		},
		computed: {
			...mapState({
				userinfo: state => state.user.userinfo
			})
		},
		methods: {
			...mapActions(['getAppInfo', 'getUserinfo']),
			...mapMutations(['SAVE_PAGE_SETTING', 'SAVE_USERINFO']),
			async login() {
				let account = null;
				// #ifdef MP-WEIXIN
				account = Account.init('WechatMiniApp')
				if (!uni.getStorageSync('openId') || !(await account.isLogin())) {
					try {
						let data = await account.login2()
						if (data) uni.setStorageSync('openId', data.openid)
						if (data.userinfo) {
							uni.setStorageSync('token', data.userinfo.token)
							this.SAVE_USERINFO(data.userinfo)
						}
					} catch (e) {}
				}
				// #endif


				// #ifdef H5
				if (this.$wxApi.isweixin()) {
					Account.init('WechatH5').login2()
				}
				// #endif
				return account;
			},

			subscribe() {
				const self = this;
				if (this.userinfo.uid) {
					const con = this.$pusher.init();
					var channel = con.subscribe('private-MESSAGECENTER-' + this.userinfo.uid);
					channel.on('message', data => {
						uni.$emit('message', data);
					})
				} else {
					setTimeout(() => {
						this.subscribe()
					}, 200);
				}
			},
		},
		onShow: function() {},
		onHide: function() {}
	}
</script>

<style lang="scss">
	@import '@/static/css/flex.scss';
	@import '@/static/css/main.scss';
	@import "@/uni_modules/uview-ui/index.scss";
	@import '@/static/css/iconfont.css';

	/* #ifdef H5 */
	page {
		background-color: #f5f5f5;
	}

	/* #endif */
	view {
		display: flex;
		flex-direction: column;
	}

	.clamp {
		overflow: hidden;
		text-overflow: ellipsis;
		display: -webkit-box;
		-webkit-line-clamp: 1;
		-webkit-box-orient: vertical;
	}

	.clamp-2 {
		-webkit-line-clamp: 2;
	}

	.clear::after {
		content: '';
		display: block;
		clear: both;
	}

	.flex-column {
		display: flex;
		flex-direction: column;
	}

	.flex-row {
		display: flex;
		flex-direction: row;
	}

	.flex-1 {
		flex: 1;
	}

	.text-center {
		text-align: center;
	}

	.u-upload,
	.u-upload__wrap {
		flex-wrap: wrap;
	}

	.u-upload__wrap {
		max-width: 100%;
	}
</style>
