<template>
	<form class='loginView'>
		<view class="input-view-item">
			<input class="input" type="digit" v-model="unitPrice" placeholder="请输入油价（元/升）"/>
		</view>
		<view class="input-view-item">
			<input class="input" type="digit" v-model="totalPrice" placeholder="加油的总价格（元）"/>
		</view>
		<view class="input-view-item">
			<input class="input" type="digit" v-model="km" placeholder="行驶里程数（公里）"/>
		</view>
		<view class="button-view-box">
			<button class="left" type="primary" @click="compute()">计算</button>
			<!--  #ifdef  MP-WEIXIN -->
			<button class="right" @click="addToHistory">记录</button>
			<!--  #endif -->
		</view>
		<view class="result-box">
			<view class="card-item">
				<text class="label">百公里油耗</text>
				<text class="value">{{oil100}}升</text>
			</view>
			<view class="card-item">
				<text class="label">一公里耗费</text>
				<text class="value">{{money1km}}元</text>
			</view>
		</view>
		<!--  #ifdef  MP-WEIXIN -->
		<view class="tip">
			提示：点击记录，可将本次的油耗数据记录下来，到历史油耗中查看爱车的油耗数据。<text @click="btnClick" class="green">{{config.btnText}}</text>
		</view>
		<!--  #endif -->
		<view @click="appreciate" class="fix-bottom">
			<ad-custom class="ad-custom" unit-id="adunit-aae810d0225a4961" ad-theme="black"></ad-custom>
			<view class="btn" v-if="config && config.showLookAd">赞赏作者</view>
		</view>
	</form>
</template>

<script>
	import NoData from '../components/no-data'
	import {
		mapState,
		mapMutations
	} from 'vuex'
	import * as common from '../../common/common'
	import * as util from '../../common/util'
	export default {
		components: {
			NoData
		},
		data() {
			return {
				unitPrice: '',
				totalPrice: '',
				km: '',
				oil100: 0,
				money1km: 0,
				historyList: [],
				loading: false,
				videoAd: null,
				oldData: null,
			};
		},
        computed: mapState(['userInfo', 'config']),
		methods: {
			...mapMutations(['getUserInfo','setStateData']),
			showTime (time) {
				return util.dateFormat(new Date(time).getTime())
			},
			compute (addToHistory) {
				let msgArr = []
				if(isNaN(Number(this.unitPrice)) || !this.unitPrice) {
					this.unitPrice = ''
					msgArr.push('油价')
				}
				if(isNaN(Number(this.totalPrice)) || !this.totalPrice) {
					this.totalPrice = ''
					msgArr.push('总价格')
				}
				if(isNaN(Number(this.km)) || !this.km) {
					this.km = ''
					msgArr.push('里程数')
				}
				if(msgArr.length) {
					uni.showToast(
							{
								title: `请输入${msgArr.join('，')}`,
								icon: 'none',
							}
					);
					return
				}
				let unitPrice = Number(this.unitPrice)
				let totalPrice = Number(this.totalPrice)
				let km = Number(this.km)
				let money1km = (totalPrice / km).toFixed(2)
				let oil100 = (this.totalPrice / this.unitPrice / km * 100).toFixed(2)
				let isSame = this.money1km === money1km && this.oil100 === oil100
				this.money1km = money1km
				this.oil100 = oil100
				if(addToHistory) {
					isSame = `${this.money1km}${this.oil100}` === this.oldData
					if (isSame) {
						uni.showToast(
								{
									title: '相同数据；无需重复记录。',
									icon: 'none'
								}
						)
						return
					}
					if (this.loading) {
						return
					}
					this.loading = true
					wx.cloud.callFunction({
						name: 'addDataToCould',
						data: {
							dbName: 'userOilCompute',
							list: [
								{
									unitPrice,
									totalPrice,
									km,
									oil100: this.oil100,
									money1km: this.money1km
								}
							]
						}
					}).then(res => {
						this.loading = false
						this.oldData = `${this.money1km}${this.oil100}`
						uni.showModal({
							title: '提示',
							content: `记录成功，可到历史油耗中查看`,
							showCancel: false,
							success: async (res3) => {
								if (res3.confirm) {
								} else if (res3.cancel) {
								}
							}
						});
					})
				}
			},
			addToHistory () {
				this.compute(true)
			},
			appreciate() {
				uni.showModal({
					title: '提示',
					showCancel: false,
					content: `观看完整视频广告，即可赞赏作者。`,
					success: (res) => {
						if (res.confirm) {
							common.showVideoAd(this.videoAd)
						} else if (res.cancel) {
						}
					}
				})
			},
			btnClick () {
				// 预览图片
				uni.previewImage({
					urls: [this.config.imgSrc],
				});
			},
			addVideoAd () {
				// 在页面onLoad回调事件中创建激励视频广告实例
				if (wx.createRewardedVideoAd) {
					this.videoAd = wx.createRewardedVideoAd({
						adUnitId: 'adunit-45bb55634ad8e65f'
					})
					this.videoAd.onLoad(() => {
					})
					this.videoAd.onError((err) => {
						console.log('videoAd.onError', err)
					})
					this.videoAd.onClose((res) => {
						// 用户点击了【关闭广告】按钮
						if (res && res.isEnded) {
							uni.showModal({
								title: '提示',
								showCancel: false,
								content: `您的赞赏，是对我们最大的肯定!`,
								success: function (res) {
									if (res.confirm) {
									} else if (res.cancel) {
									}
								}
							})
						} else {
							// 播放中途退出，不下发游戏奖励
						}
					})
				}
			}
		},
		// 加了这个页面才可以被分享
		onShareAppMessage () {
		},
		async onLoad() {
			// 挂载视频广告
			this.addVideoAd()
		}
	}
</script>

<style lang="stylus" scoped>
	@import "../../uni.styl"
	.green
		color $uni-color-success
	.input-view-item
		border-bottom 1px solid $uni-border-color
		padding 10px
	.result-box
		display flex
		justify-content space-between
		flex-direction column
		border-radius 10px
		background-color $uni-list-item-hover-color
		margin 0 10px
	.card-item
		display flex
		align-items center
		justify-content space-between
		padding 10px 20px
		.label
			margin-right 5px
			font-size 20px
		.value
			font-size 20px
			color $uni-color-success
		&:last-child
			border-top 1px solid $uni-list-item-color
	.fix-bottom
		position fixed
		bottom 0
		left 0
		right 0
		padding 10px 0
		display block
		.btn
			color $uni-color-success
			width 100%
			display block
			font-size 12px
			text-align center
			text-decoration underline
			padding-top 5px
	.button-view-box
		display flex
		padding 10px
		.left
			flex 3
		.right
			margin-left 20px
			flex 1
	.tip
		display block
		padding 5px 10px
		font-size 12px
	.ad-custom
		width 50%
		margin 0 auto
</style>
