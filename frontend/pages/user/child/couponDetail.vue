<template>

	<u-popup :show="show" @open="open" @close="close" class="popup-content" >
		<view class="">
			<view class="content" v-if="couponType == 2">
				<text class="h3">限以下商品使用</text>
				<view class="item" v-for="(item, index) in detail" :key="index">
					<view class="product u-flex-row" @click="goProduct(item)">
						<view class="image">
							<image v-if="item.image && item.image[0]" :src="item.image && $tools.fixurl(item.image[0])"
								mode="aspectFill"></image>
						</view>
						<view class="desc u-flex-row">
							<text class="title u-line-3">{{item.title}}</text>
							<text class="price">￥{{item.price}}</text>
						</view>
					</view>
				</view>
			</view>
			<view class="content category-content" v-if="couponType == 1">
				<text class="h3">限以下分类使用</text>
				<view class="item" v-for="(item, index) in detail" :key="index" @click="goCategory(item)">
					<text>{{item.name}}</text>
				</view>
			</view>
		</view>
	</u-popup>
</template>

<script>
	import {
		mapState
	} from 'vuex';
	export default {
		computed: {
			...mapState({
				user: state => state.user.userinfo
			})
		},
		data() {
			return {
				show: false
			}
		},
		props: {
			couponType: {
				type: String | Number,
				default: 0
			},
			detail: {
				type: Array
			}
		},
		methods: {
			goProduct(item) {
				uni.navigateTo({
					url: '/pages/product/product?id=' + item.id
				})
			},
			goCategory(item) {
				uni.navigateTo({
					url: '/pages/product/list?cat_id=' + item.id
				})
			},
			open() {
				this.show = true
			},
			close() {
				this.show = false
			}
		}
	}
</script>

<style lang="scss" scoped>
	.h3 {
		color: #888;
		display: flex;
		flex: 1;
		justify-content: center;
		align-items: center;
		border-bottom: #eee;
		font-size: 28upx;
		margin: 20upx;
	}

	.popup-content {
		font-size: 26upx;

		.content {
			max-height: 60vh;
			overflow-y: scroll;
			background-color: #fff;
		}

		.item {
			display: flex;
			flex-direction: column;
		}

		.product {
			display: flex;
			height: 140upx;
			margin: 10upx 0;

			.image {
				width: 100%;
				max-width: 140upx;
				height: 140upx;
				border-radius: 3px;
				overflow: hidden;

				image {
					width: 100%;
					height: 100%;
					opacity: 1;
				}
			}

			.desc {
				display: flex;
				flex: 1;
				line-height: 150%;
				text-align: left;
				padding: 10upx;
				justify-content: space-between;

				.price {
					color: #fa436a;
					display: flex;
					justify-content: center;
					align-items: center;
				}
			}
		}

		.category-content {
			max-height: 40vh;
			overflow-y: scroll;

			.item {
				font-size: 30upx;
				text-align: center;
				margin: 30upx 0;
			}
		}
	}
</style>
