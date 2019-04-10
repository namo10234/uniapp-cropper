<template>
	<view class="wx-content-info">
		<view class='cropper-content'>
			<view v-if="isShowImg" class="wx-corpper" :style="{width:cropperInitW+'px',height:cropperInitH+'px'}" style="background:#000">
				<view class="wx-corpper-content" :style="{width:cropperW+'px',height:cropperH+'px',left:cropperL+'px',top:cropperT+'px'}">
					<image :src="imageSrc" v-if="imageSrc" :style="{width:cropperW+'px',height:cropperH+'px'}"></image>
					<view class="wx-corpper-crop-box" @touchstart="contentStartMove" @touchmove="contentMoveing" 
					 :style="{left:cutL+'px',top:cutT+'px',right:cutR+'px',bottom:cutB+'px'}">
						<view class="wx-cropper-view-box">
							<view class="wx-cropper-dashed-h"></view>
							<view class="wx-cropper-dashed-v"></view>
							<view class="wx-cropper-line-t" data-drag="top" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-line-r" data-drag="right" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-line-b" data-drag="bottom" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-line-l" data-drag="left" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-point point-t" data-drag="top" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-point point-tr" data-drag="topTight"></view>
							<view class="wx-cropper-point point-r" data-drag="right" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-point point-rb" data-drag="rightBottom" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-point point-b" data-drag="bottom" @touchstart.stop="dragStart" @touchmove.stop="dragMove"
							 @touchend.stop="dragEnd"></view>
							<view class="wx-cropper-point point-bl" data-drag="bottomLeft"></view>
							<view class="wx-cropper-point point-l" data-drag="left" @touchstart.stop="dragStart" @touchmove.stop="dragMove"></view>
							<view class="wx-cropper-point point-lt" data-drag="leftTop"></view>
						</view>
					</view>
				</view>
			</view>
		</view>
		<view class='cropper-config'>
			<button type="primary reverse" @tap="getImage" style='margin-top: 30upx;'> 选择图片 </button>
			<button type="primary" @tap="getImageInfo" style='margin-top: 30upx;'> 点击生成图片 </button>
		</view>
		<canvas canvas-id="myCanvas" :style="{ width:qualityWidth+'px',height:qualityWidth/innerAspectRadio+'px'}" style="position:absolute;border:1upx solid red;top:-9999upx;left:-9999upx;"></canvas>
	</view>
</template>

<script>
	var SCREEN_WIDTH = uni.upx2px(750)
	var PAGE_X, // 手按下的x位置
		PAGE_Y, // 手按下y的位置
		PR = uni.getSystemInfoSync().pixelRatio, // dpi
		T_PAGE_X, // 手移动的时候x的位置
		T_PAGE_Y, // 手移动的时候Y的位置
		CUT_L, // 初始化拖拽元素的left值
		CUT_T, // 初始化拖拽元素的top值
		CUT_R, // 初始化拖拽元素的
		CUT_B, // 初始化拖拽元素的
		CUT_W, // 初始化拖拽元素的宽度
		CUT_H, //  初始化拖拽元素的高度
		IMG_RATIO, // 图片比例
		IMG_REAL_W, // 图片实际的宽度
		IMG_REAL_H, // 图片实际的高度
		DRAFG_MOVE_RATIO = uni.upx2px(750) / uni.getSystemInfoSync().windowWidth, //移动时候的比例,
		INIT_DRAG_POSITION = uni.upx2px(200), // 初始化屏幕宽度和裁剪区域的宽度之差，用于设置初始化裁剪的宽度
		DRAW_IMAGE_W // 设置生成的图片宽度

	export default {
		data() {
			return {
				// 之后可以动态替换
				imageSrc: '/static/logo.png',

				// 是否显示图片(在图片加载完成之后设置为true)
				isShowImg: false,

				// 初始化的宽高
				cropperInitW: SCREEN_WIDTH,
				cropperInitH: SCREEN_WIDTH,

				// 动态的宽高
				cropperW: SCREEN_WIDTH,
				cropperH: SCREEN_WIDTH,

				// 动态的left top值
				cropperL: 0,
				cropperT: 0,

				// 图片缩放值
				scaleP: 0,

				// 裁剪框 宽高
				cutL: 0,
				cutT: 0,
				cutB: SCREEN_WIDTH,
				cutR: '100%',
				qualityWidth: DRAW_IMAGE_W,
				innerAspectRadio: DRAFG_MOVE_RATIO
			}
		},
		onReady() {
			this.loadImage();
		},

		methods: {
			 //选择本地图片
			getImage: function() {
				var _this = this
				uni.chooseImage({
					success: function(res) {
						_this.imageSrc = res.tempFilePaths[0],
						_this.loadImage();
					},
				})
			},


			loadImage: function() {
				var _this = this
				uni.showLoading({
					title: '图片加载中...',
				})

				uni.getImageInfo({
					src: _this.imageSrc,
					success: function(res) {
						DRAW_IMAGE_W = IMG_REAL_W = res.width
						IMG_REAL_H = res.height
						IMG_RATIO = IMG_REAL_W / IMG_REAL_H
						let minRange = IMG_REAL_W > IMG_REAL_H ? IMG_REAL_W : IMG_REAL_H
						INIT_DRAG_POSITION = minRange > INIT_DRAG_POSITION ? INIT_DRAG_POSITION : minRange
						// 根据图片的宽高显示不同的效果   保证图片可以正常显示
						if (IMG_RATIO >= 1) {
							_this.cropperW = SCREEN_WIDTH,
								_this.cropperH = SCREEN_WIDTH / IMG_RATIO,
								// 初始化left right
								_this.cropperL = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH) / 2),
								_this.cropperT = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH / IMG_RATIO) / 2),
								_this.cutL = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH + INIT_DRAG_POSITION) / 2),
								_this.cutT = Math.ceil((SCREEN_WIDTH / IMG_RATIO - (SCREEN_WIDTH / IMG_RATIO - INIT_DRAG_POSITION)) / 2),
								_this.cutR = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH + INIT_DRAG_POSITION) / 2),
								_this.cutB = Math.ceil((SCREEN_WIDTH / IMG_RATIO - (SCREEN_WIDTH / IMG_RATIO - INIT_DRAG_POSITION)) / 2),
								// 图片缩放值
								_this.scaleP = IMG_REAL_W / SCREEN_WIDTH,
								_this.qualityWidth = DRAW_IMAGE_W,
								_this.innerAspectRadio = IMG_RATIO
						} else {
							_this.cropperW = SCREEN_WIDTH * IMG_RATIO,
								_this.cropperH = SCREEN_WIDTH,
								// 初始化left right
								_this.cropperL = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH * IMG_RATIO) / 2),
								_this.cropperT = Math.ceil((SCREEN_WIDTH - SCREEN_WIDTH) / 2),

								_this.cutL = Math.ceil((SCREEN_WIDTH * IMG_RATIO - (SCREEN_WIDTH * IMG_RATIO)) / 2),
								_this.cutT = Math.ceil((SCREEN_WIDTH - INIT_DRAG_POSITION) / 2),
								_this.cutB = Math.ceil((SCREEN_WIDTH - INIT_DRAG_POSITION) / 2),
								_this.cutR = Math.ceil((SCREEN_WIDTH * IMG_RATIO - (SCREEN_WIDTH * IMG_RATIO)) / 2),
								// 图片缩放值
								_this.scaleP = IMG_REAL_W / SCREEN_WIDTH,
								_this.qualityWidth = DRAW_IMAGE_W,
								_this.innerAspectRadio = IMG_RATIO
						}
						_this.isShowImg = true
						uni.hideLoading()
					}
				})
			},


			contentStartMove(e) {
				PAGE_X = e.touches[0].pageX
				PAGE_Y = e.touches[0].pageY
				console.log('contentStartMove')
			},

			contentMoveing(e) {
				var _this = this
				var dragLengthX = (PAGE_X - e.touches[0].pageX) * DRAFG_MOVE_RATIO
				var dragLengthY = (PAGE_Y - e.touches[0].pageY) * DRAFG_MOVE_RATIO

				// 左移右移
				if (dragLengthX > 0) {
					if (this.cutL - dragLengthX < 0) {dragLengthX = this.cutL}
				} else {
					if (this.cutR + dragLengthX < 0) {dragLengthX = -this.cutR}
				}

				// 上移下移
				if (dragLengthY > 0) {
					if (this.cutT - dragLengthY < 0) {dragLengthY = this.cutT}
				} else {
					if (this.cutB + dragLengthY < 0) {dragLengthY = -this.cutB}
				}
				this.cutL= this.cutL - dragLengthX,
				this.cutT= this.cutT - dragLengthY,
				this.cutR= this.cutR + dragLengthX,
				this.cutB= this.cutB + dragLengthY
				PAGE_X = e.touches[0].pageX
				PAGE_Y = e.touches[0].pageY
			},

			getImageInfo() {
				var _this = this
				uni.showLoading({
					title: '图片生成中...',
				})
				// 将图片写入画布
				const ctx = uni.createCanvasContext('myCanvas')
				ctx.drawImage(_this.imageSrc, 0, 0, IMG_REAL_W, IMG_REAL_H);
				ctx.draw(true, () => {
					// 获取画布要裁剪的位置和宽度   均为百分比 * 画布中图片的宽度    保证了在微信小程序中裁剪的图片模糊  位置不对的问题
					var canvasW = ((_this.cropperW - _this.cutL - _this.cutR) / _this.cropperW) * IMG_REAL_W
					var canvasH = ((_this.cropperH - _this.cutT - _this.cutB) / _this.cropperH) * IMG_REAL_H
					var canvasL = (_this.cutL / _this.cropperW) * IMG_REAL_W
					var canvasT = (_this.cutT / _this.cropperH) * IMG_REAL_H
					// 生成图片
					uni.canvasToTempFilePath({
						x: canvasL,
						y: canvasT,
						width: canvasW,
						height: canvasH,
						destWidth: canvasW,
						destHeight: canvasH,
						quality: 0.5,
						canvasId: 'myCanvas',
						success: function(res) {
							uni.hideLoading()
							// 成功获得地址的地方
							uni.previewImage({
								current: '', // 当前显示图片的http链接
								urls: [res.tempFilePath] // 需要预览的图片http链接列表
							})
						}
					})
				})
			},


			dragStart(e) {
				T_PAGE_X = e.touches[0].pageX
				T_PAGE_Y = e.touches[0].pageY
				CUT_L = this.cutL
				CUT_R = this.cutR
				CUT_B = this.cutB
				CUT_T = this.cutT
			},


			dragMove(e) {
				var _this = this
				var dragType = e.target.dataset.drag
				switch (dragType) {
					case 'right':
						var dragLength = (T_PAGE_X - e.touches[0].pageX) * DRAFG_MOVE_RATIO
						if (CUT_R + dragLength < 0) dragLength = -CUT_R
							this.cutR = CUT_R + dragLength
						break;
					case 'left':
						var dragLength = (T_PAGE_X - e.touches[0].pageX) * DRAFG_MOVE_RATIO
						if (CUT_L - dragLength < 0) dragLength = CUT_L
						if ((CUT_L - dragLength) > (this.cropperW - this.utR)) dragLength = CUT_L - (this.cropperW - this.cutR)
							this.cutL = CUT_L - dragLength
						break;
					case 'top':
						var dragLength = (T_PAGE_Y - e.touches[0].pageY) * DRAFG_MOVE_RATIO
						if (CUT_T - dragLength < 0) dragLength = CUT_T
						if ((CUT_T - dragLength) > (this.cropperH - this.cutB))
							 dragLength = CUT_T - (this.cropperH - this.cutB)
							 this.cutT = CUT_T - dragLength
						break;
					case 'bottom':
						var dragLength = (T_PAGE_Y - e.touches[0].pageY) * DRAFG_MOVE_RATIO
						if (CUT_B + dragLength < 0) dragLength = -CUT_B
							this.cutB = CUT_B + dragLength
						break;
					case 'rightBottom':
						var dragLengthX = (T_PAGE_X - e.touches[0].pageX) * DRAFG_MOVE_RATIO
						var dragLengthY = (T_PAGE_Y - e.touches[0].pageY) * DRAFG_MOVE_RATIO
						if (CUT_B + dragLengthY < 0) 
						dragLengthY = -CUT_B
						if (CUT_R + dragLengthX < 0) 
						dragLengthX = -CUT_R
						this.cutB = CUT_B + dragLengthY,
						this.cutR = CUT_R + dragLengthX
						break;
					default:
						break;
				}
			}
		}
	}
</script>

<style>
	/* pages/wx-cropper/index.wxss */
	.wx-content-info {
		position: fixed;
		top: 0;
		left: 0;
		right: 0;
		bottom: 0;
		display: block;
		align-items: center;
		flex-direction: column;
	}

	.cropper-config {
		padding: 20upx 40upx;
	}

	.cropper-content {
		min-height: 750upx;
		width: 100%;
	}

	.wx-corpper {
		position: relative;
		overflow: hidden;
		-webkit-user-select: none;
		-moz-user-select: none;
		-ms-user-select: none;
		user-select: none;
		-webkit-tap-highlight-color: transparent;
		-webkit-touch-callout: none;
		box-sizing: border-box;
	}

	.wx-corpper-content {
		position: relative;
	}

	.wx-corpper-content image {
		display: block;
		width: 100%;
		min-width: 0 !important;
		max-width: none !important;
		height: 100%;
		min-height: 0 !important;
		max-height: none !important;
		image-orientation: 0deg !important;
		margin: 0 auto;
	}

	/* 移动图片效果 */
	.wx-cropper-drag-box {
		position: absolute;
		top: 0;
		right: 0;
		bottom: 0;
		left: 0;
		cursor: move;
		background: rgba(0, 0, 0, 0.6);
		z-index: 1;
	}

	/* 内部的信息 */
	.wx-corpper-crop-box {
		position: absolute;
		background: rgba(255, 255, 255, 0.3);
		z-index: 2;
	}

	.wx-corpper-crop-box .wx-cropper-view-box {
		position: relative;
		display: block;
		width: 100%;
		height: 100%;
		overflow: visible;
		outline: 1px solid #69f;
		outline-color: rgba(102, 153, 255, .75)
	}

	/* 横向虚线 */
	.wx-cropper-dashed-h {
		position: absolute;
		top: 33.33333333%;
		left: 0;
		width: 100%;
		height: 33.33333333%;
		border-top: 1px dashed rgba(255, 255, 255, 0.5);
		border-bottom: 1px dashed rgba(255, 255, 255, 0.5);
	}

	/* 纵向虚线 */
	.wx-cropper-dashed-v {
		position: absolute;
		left: 33.33333333%;
		top: 0;
		width: 33.33333333%;
		height: 100%;
		border-left: 1px dashed rgba(255, 255, 255, 0.5);
		border-right: 1px dashed rgba(255, 255, 255, 0.5);
	}

	/* 四个方向的线  为了之后的拖动事件*/
	.wx-cropper-line-t {
		position: absolute;
		display: block;
		width: 100%;
		background-color: #69f;
		top: 0;
		left: 0;
		height: 1px;
		opacity: 0.1;
		cursor: n-resize;
	}

	.wx-cropper-line-t::before {
		content: '';
		position: absolute;
		top: 50%;
		right: 0upx;
		width: 100%;
		-webkit-transform: translate3d(0, -50%, 0);
		transform: translate3d(0, -50%, 0);
		bottom: 0;
		height: 41upx;
		background: transparent;
		z-index: 11;
	}

	.wx-cropper-line-r {
		position: absolute;
		display: block;
		background-color: #69f;
		top: 0;
		right: 0px;
		width: 1px;
		opacity: 0.1;
		height: 100%;
		cursor: e-resize;
	}

	.wx-cropper-line-r::before {
		content: '';
		position: absolute;
		top: 0;
		left: 50%;
		width: 41upx;
		-webkit-transform: translate3d(-50%, 0, 0);
		transform: translate3d(-50%, 0, 0);
		bottom: 0;
		height: 100%;
		background: transparent;
		z-index: 11;
	}

	.wx-cropper-line-b {
		position: absolute;
		display: block;
		width: 100%;
		background-color: #69f;
		bottom: 0;
		left: 0;
		height: 1px;
		opacity: 0.1;
		cursor: s-resize;
	}

	.wx-cropper-line-b::before {
		content: '';
		position: absolute;
		top: 50%;
		right: 0upx;
		width: 100%;
		-webkit-transform: translate3d(0, -50%, 0);
		transform: translate3d(0, -50%, 0);
		bottom: 0;
		height: 41upx;
		background: transparent;
		z-index: 11;
	}

	.wx-cropper-line-l {
		position: absolute;
		display: block;
		background-color: #69f;
		top: 0;
		left: 0;
		width: 1px;
		opacity: 0.1;
		height: 100%;
		cursor: w-resize;
	}

	.wx-cropper-line-l::before {
		content: '';
		position: absolute;
		top: 0;
		left: 50%;
		width: 41upx;
		-webkit-transform: translate3d(-50%, 0, 0);
		transform: translate3d(-50%, 0, 0);
		bottom: 0;
		height: 100%;
		background: transparent;
		z-index: 11;
	}

	.wx-cropper-point {
		width: 5upx;
		height: 5upx;
		background-color: #69f;
		opacity: .75;
		position: absolute;
		z-index: 3;
	}

	.point-t {
		top: -3upx;
		left: 50%;
		margin-left: -3upx;
		cursor: n-resize;
	}

	.point-tr {
		top: -3upx;
		left: 100%;
		margin-left: -3upx;
		cursor: n-resize;
	}

	.point-r {
		top: 50%;
		left: 100%;
		margin-left: -3upx;
		margin-top: -3upx;
		cursor: n-resize;
	}

	.point-rb {
		left: 100%;
		top: 100%;
		-webkit-transform: translate3d(-50%, -50%, 0);
		transform: translate3d(-50%, -50%, 0);
		cursor: n-resize;
		width: 16upx;
		height: 16upx;
		background-color: #69f;
		position: absolute;
		z-index: 1112;
		opacity: 1;
	}

	.point-rb::before {
		content: '';
		position: absolute;
		top: -12upx;
		left: -12upx;
		right: -12upx;
		bottom: -12upx;
	}

	.point-b {
		left: 50%;
		top: 100%;
		margin-left: -3upx;
		margin-top: -3upx;
		cursor: n-resize;
	}

	.point-bl {
		left: 0%;
		top: 100%;
		margin-left: -3upx;
		margin-top: -3upx;
		cursor: n-resize;
	}

	.point-l {
		left: 0%;
		top: 50%;
		margin-left: -3upx;
		margin-top: -3upx;
		cursor: n-resize;
	}

	.point-lt {
		left: 0%;
		top: 0%;
		margin-left: -3upx;
		margin-top: -3upx;
		cursor: n-resize;
	}

	/* 裁剪框预览内容 */
	.wx-cropper-viewer {
		position: relative;
		width: 100%;
		height: 100%;
		overflow: hidden;
	}

	.wx-cropper-viewer image {
		position: absolute;
		z-index: 2;
	}
</style>
