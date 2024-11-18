<template>
	<view id="app">
		<view class="game-top">
			<view class="view-center">
				<view class="top-nav">
					<view>当前积分</view>
					<view class="number">{{ score }}</view>
				</view>
				<view class="top-nav">
					<view>最高积分</view>
					<view class="number">{{ maxScore }}</view>
				</view>
			</view>
		</view>

		<view class="game-main" @touchstart="handleTouchstart" @touchend="handleTouchend">
			<view class="main-bor clearfix">
				<view class="item bgItem" v-for="(item, index) in nums" :key="index">
					<!-- 显示层 -->
					<view style="z-index: 500;" :animation="animationDataNum[index]" class="item-li view-center"
						:class="`c${item}`">
						{{ item }}
					</view>
					<!-- 移动层 -->
					<view style="z-index: 499;" :animation="animationData[index]" class="item-li view-center"
						:class="`c${copyNums[index]}`">
						{{ copyNums[index] }}
					</view>
				</view>
			</view>
		</view>
		<button v-if="!isOnGame" @click="startGame()">开始</button>
		<button v-if="isOnGame" @click="handleRestGame()">重置</button>

	</view>
</template>

<script setup>
	import {
		ref,
		reactive,
		nextTick,
		onMounted
	} from 'vue';
	import {
		onLoad
	} from '@dcloudio/uni-app';

	//合并动画
	const animationData = ref([]);
	//移动动画
	const animationDataNum = ref([]);
	const animationRef = ref();

	const numPox = ref([]);
	const start = ref({
		x: 0,
		y: 0
	}); //记录移动端触摸起始点
	const nums = ref([]); //记录16个框格的属性
	const copyNums = ref([]); //移动格子
	//是否移动中
	const isMove = ref(false);
	const score = ref(0); //当前积分
	const maxScore = ref(0); //最高积分
	//是否游戏中
	const isOnGame = ref(false);

	onLoad(() => {
		nums.value = Array(16).join('-').split('-');
		init();

	})
	onMounted(() => {
		nextTick(() => {
			let info = uni.createSelectorQuery().selectAll(".bgItem");
			info.boundingClientRect((data) => { //data - 各种参数
				numPox.value = data;
			}).exec()
		})
	})

	//初始化游戏数据
	const init = () => {
		maxScore.value = uni.getStorageSync('maxScore');
		startGame();
	}

	//获取对应的位置
	const getIndexPos = (index) => {
		return numPox.value[index];
	}

	//重置游戏
	const handleRestGame = () => {
		uni.showModal({
			title: '确定重置',
			content: '确定重置当前游戏！',
			success: function(res) {
				if (res.confirm) {
					startGame();
				}
			}
		});
	}

	//获取空白数组
	const blankIndex = () => {
		let arr = [];
		nums.value.forEach((i, j) => {
			i === '' && arr.push(j);
		});
		return arr;
	}

	/*在一个随机的空白位添加数字*/
	const randomAdd = (count = 1) => {
		let arr = shuffle(blankIndex());
		let random = [2, 2, 2, 2, 2, 2, 2, 4, 4, 4, 8]
		while (count--) {
			let arrIndex = arr.pop();
			//延时100毫秒添加
			setTimeout(_ => {
				nums.value.splice(arrIndex, 1, random[Math.floor(Math.random() * 10)]);
				//计算分数
				getScore();
			}, 100);
		}
	}

	/*打乱数组*/
	const shuffle = (arr) => {
		let l = arr.length,
			j;
		while (l--) {
			j = parseInt(Math.random() * l);
			[arr[l], arr[j]] = [arr[j], arr[l]]
		}
		return arr;
	}

	//重置游戏
	const reset = () => {
		score.value = 0;
		nums.value = Array(16).join('-').split('-');
		copyNums.value = [...nums.value];
		let animation = uni.createAnimation({
			timingFunction: "linear",
		})
		//还原动画
		for (let a = 0; a < 16; a++) {
			animation.opacity(1).step({
				duration: 0
			});
			animationDataNum.value[a] = animation.export();
			animation.opacity(1).translateY(0).translateX(0).step({
				duration: 0
			});
			animationData.value[a] = animation.export();
		}
	}

	//开始触摸
	const handleTouchstart = (e) => {
		start.value.x = e.changedTouches[0].pageX;
		start.value.y = e.changedTouches[0].pageY;
	}
	//触摸结束
	const handleTouchend = (e) => {
		let curPoint = e.changedTouches[0],
			x = curPoint.pageX - start.value.x,
			y = curPoint.pageY - start.value.y,
			xx = Math.abs(x),
			yy = Math.abs(y),
			i = 0;
		//移动范围太小 不处理
		//0左  2右  3下 1 上
		if (xx < 50 && yy < 50) return;
		if (xx >= yy) { //横向滑动
			i = x < 0 ? 0 : 2;
		} else { //纵向滑动
			i = y < 0 ? 3 : 1;
		}
		handle(i);
	}
	//触摸完成移动方块
	const handle = (i) => {
		if (!isMove.value) {
			copyNums.value = [...nums.value];
			move(i);
		}

	}
	/*把数组矩阵，旋转n次，旋转后统一按照左移处理*/
	const transposeMatrix = (arr, n) => {
		n = n % 4;
		if (n === 0) return arr;
		let l = arr.length,
			d = Math.sqrt(l),
			tmp = [];
		for (let i = 0; i < d; i += 1)
			for (let j = 0; j < d; j += 1)
				tmp[d - i - 1 + j * d] = arr[i * d + j];
		if (n > 1) tmp = transposeMatrix(tmp, n - 1);
		return tmp;
	}

	/*移动方块 i:旋转次数 */
	const move = (i) => {
		isMove.value = true;
		let indexs = transposeMatrix(Object.keys(String(Array(17))), i); //记录旋转前的位置索引
		let tmp = transposeMatrix([...nums.value], i); //矩阵旋转
		let hasMove = false; //判断是偶有移动
		let hasCombin = {}; //记录合并索引
		tmp.forEach((item, index) => {
			let newIndex = 0, //方块挪动后的索引 （转换后的索引）
				oldIndex = indexs[index] - 0, //换算到转换前的索引
				thisMoved = false, //此方块有数字，且被移动了 标记  需要应用动画
				combinNum = 0; //方块若有合并，记录合并后的数字
			while (index % 4 && item !== '') {
				if (tmp[index - 1] === '') { //当前位置的前一位置为空,交换俩位置
					tmp[index - 1] = item;
					tmp[index] = '';
					hasMove = true;
					thisMoved = true;
					newIndex = index - 1;
				} else if (tmp[index - 1] === item && !hasCombin[index] && !hasCombin[index - 1] && item <
					2048) {
					//当前位置与前一位置数字相同，合并到前一位置，然后清空当前位置
					item *= 2;
					tmp[index - 1] = item;
					tmp[index] = '';
					hasMove = true;
					thisMoved = true;
					combinNum = item;
					hasCombin[index - 1] = true; //记录合并位置
					newIndex = index - 1;
				} else {
					break;
				}
				index--;
			}
			thisMoved && moveNode(oldIndex, indexs[newIndex], combinNum, i);
		});
		//移动动画完成后还原
		setTimeout(_ => {
			nums.value = [...transposeMatrix(tmp, 4 - i)]; //转置回去，把数据还给nums
			let animation = uni.createAnimation({
				timingFunction: "step-start",
			})

			animationRef.value = animation;
			//显示显示层
			for (let a = 0; a < 16; a++) {
				animationRef.value.opacity(1).step({
					duration: 0
				});
				animationDataNum.value[a] = animationRef.value.export();
			}
			//设置合并动画
			Object.keys(hasCombin).forEach(key => {
				animationRef.value.opacity(1).step({
					duration: 0
				});
				animationRef.value.scale(0.8).step({
					duration: 50
				});
				animationRef.value.scale(1).step({
					duration: 100
				});
				animationDataNum.value[indexs[key]] = animationRef.value.export();
			});
			//还原移动层
			for (let a = 0; a < 16; a++) {
				animationRef.value.opacity(0).translateY(0).translateX(0).step({
					duration: 0
				});
				animationData.value[a] = animationRef.value.export();
			}
			if (hasMove) {
				randomAdd();
			} else {
				gameOver();
			}
			setTimeout(function() {
				let animation = uni.createAnimation({
					timingFunction: "step-start",
				})
				//重置移动层数据
				copyNums.value = [...nums.value];
				for (let a = 0; a < 16; a++) {
					animation.opacity(1).step({
						duration: 0
					});
					animationData.value[a] = animation.export();
				}
				isMove.value = false;
			}, 50);
		}, 152);
	}


	//索引index的元素移动到nextIndex
	const moveNode = (index, nextIndex, combinNum, i) => {
		let nowPos = getIndexPos(index);
		let nextPos = getIndexPos(nextIndex);
		let animation = uni.createAnimation({
			timingFunction: "ease",
			delay: 0
		})

		animationRef.value = animation;
		//移动层移动
		if (i === 0 || i === 2) {
			animationRef.value.translateX(nextPos.left - nowPos.left).step({
				duration: 150
			});
		} else {
			animationRef.value.translateY(nextPos.top - nowPos.top).step({
				duration: 150
			});
		}
		animationData.value[index] = animationRef.value.export();
		//隐藏显示层
		animationRef.value.opacity(0).step({
			duration: 0
		});
		animationDataNum.value[index] = animationRef.value.export();
	}

	//开始游戏
	const startGame = () => {
		//重置分数-数据
		reset()
		//添加方块
		randomAdd(2)
		//游戏标识
		isOnGame.value = true;
	}
	//判断游戏结束
	const gameOver = () => {
		let isOver = true;
		for (let index = 0; index < nums.value.length; index++) {
			let item = nums.value[index] || 0;
			if (item == 0) {
				isOver = false;
				break;
			}
			if (item >= 2048) {
				continue;
			}
			let mod = index % 4;
			if (mod == 0) {
				if (nums.value[index - 4] == item || nums.value[index + 4] == item || nums.value[index + 1] == item) {
					isOver = false;
					break;
				}
			} else if (mod == 3) {
				if (nums.value[index - 4] == item || nums.value[index + 4] == item || nums.value[index - 1] == item) {
					isOver = false;
					break;
				}
			} else {
				if (nums.value[index - 4] == item || nums.value[index + 4] == item || nums.value[index - 1] == item ||
					nums
					.value[index + 1] == item) {
					isOver = false;
					break;
				}
			}

		}
		if (isOver) {
			//游戏标识
			isOnGame.value = false;
			uni.showModal({
				title: '游戏结束',
				content: '你已经无路可走了！',
				success: function(res) {
					reset();
				}
			});
		}
	}


	const getScore = () => {
		let cScore = 0;
		nums.value.forEach((item) => {
			if (item !== "") {
				cScore = cScore + Number(item);
			}
		})
		score.value = cScore;
		if (cScore > maxScore.value) {
			maxScore.value = cScore;
			uni.setStorageSync('maxScore', cScore);
		}
	}
</script>

<style lang="scss" scoped>
	body {
		overflow: hidden;
	}

	#app {
		width: 100%;
		position: relative;
		font-family: 'Inknut Antiqua', 'Clear Sans', 'Helvetica Neue', Arial, sans-serif;
	}

	.game-top {
		margin: 0rpx 48rpx 48rpx 48rpx;
		background: #ffe4b3;
		border-radius: 20rpx;
		padding: 50rpx 50rpx 100rpx 50rpx;

		.top-nav {
			padding-top: 10rpx;
			width: 178rpx;
			height: 90rpx;
			background: #f96e51;
			border: 4rpx solid rgba(177, 118, 78, 1);
			border-radius: 14rpx;
			text-align: center;
			font-size: 24rpx;
			color: rgba(255, 255, 255, 1.0);
			margin: 0 10rpx;

			.number {
				padding-top: 4rpx;
				color: #ffe4b3;
				font-size: 28rpx;
				font-weight: 600;
			}
		}
	}

	.game-n {
		margin-top: 20rpx;
		color: #f96e51;

		.view-flex__item:last-child {
			text-align: right;
		}
	}

	.game-slider {
		border: 4rpx solid #7c4d28;
		background: #5b3610;
		height: 28rpx;
		margin-top: 10rpx;
		border-radius: 20rpx;
		position: relative;

		.slider-w {
			background: #ffca00;
			height: 20rpx;
			border-radius: 20rpx;
			max-width: 100% !important;
		}

		.slider-p {
			position: absolute;
			top: -5rpx;
			left: 0;
			width: 100%;

			.view-flex__item {
				text-align: right;
				position: relative;

				.p-bx {
					width: 76rpx;
					text-align: center;
					position: absolute;
					top: 50rpx;
					right: -24rpx;

					img {
						width: 76rpx;
					}
				}
			}
		}
	}

	.game-main {
		margin: 0 48rpx;
		border-radius: 20rpx;
		overflow: hidden;

		.main-bor::after {
			content: "";
			display: table;
			clear: both;
		}

		.main-bor {
			border: 10rpx solid #f8f3e8;
			;
			background: rgba(177, 118, 78, 0.3);
			border-radius: 20rpx;
			padding: 5rpx;

			.item {
				float: left;
				width: calc(25% - 10rpx);
				margin: 5rpx;
				padding-bottom: calc(25% - 10rpx);
				background: #f8f3e8;
				position: relative;

				.item-li {
					position: absolute;
					width: 100%;
					height: 100%;
					font-size: 80rpx;
					font-weight: bold;
					color: #ffffff;

					&.c2 {
						background: #ffe4b3;
						color: #986d0f;
					}

					&.c4 {
						background: #f9ce7f;
						color: #986d0f;
					}

					&.c8 {
						background: #fcbd3b;
						color: #986d0f;
					}

					&.c16 {
						background: #f98e78;
					}

					&.c32 {
						background: #f96e51;
					}

					&.c64 {
						background: #f599b2;
					}

					&.c128 {
						background: #ec6187;
					}

					&.c256 {
						background: #1b67b3;
					}

					&.c512 {
						background: #3884cf;
					}

					&.c1024 {
						background: #9c79e0;
						font-size: 60rpx;
					}

					&.c2048 {
						background: #6d41c1;
						font-size: 60rpx;
					}
				}
			}
		}
	}

	::v-deep .rule-w {
		.u-popup__content {
			width: 80%;
			background: none;
		}

		.u-icon__icon {
			color: #fff !important;
		}

		.rule-main {
			background: #bc7638;
			padding: 20rpx;
			border-radius: 10rpx;
			max-height: 50vh;
			overflow-y: auto;
			font-size: 24rpx;

			.title {
				padding-bottom: 10rpx;
			}

			.text {
				padding-bottom: 10rpx;
				line-height: 44rpx;
			}
		}
	}

	.rule-open {
		padding: 10rpx;
		text-align: center;
	}

	.rule-tip {
		width: 200rpx;

		img {
			vertical-align: middle;
			margin: -4rpx 8rpx 0 0;
		}
	}


	#app button {
		width: 100px;
		height: 50px;
		line-height: 50px;
		font-size: 20px;
		display: block;
		cursor: pointer;
		margin: 20px auto;
		background-color: #2daebf;
		border-color: #238896;
		color: white;
		box-shadow: 0 -2px 0 rgba(0, 0, 0, 0.2) inset;
		border-radius: 4px;
		transition: background-color 300ms ease-out;
		animation: bluePulse 2s 4s infinite;
	}

	.uni-stat-tooltip {
		max-width: 400rpx;
		max-height: 400rpx;
		overflow: scroll;
		word-break: break-word;
	}

	.view-center {
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.view-flex {
		display: -webkit-box;
		display: -webkit-flex;
		display: flex
	}

	.view-flex__item {
		-webkit-box-flex: 1;
		-webkit-flex: 1;
		flex: 1
	}

	@keyframes zy {
		10% {
			transform: rotate(15deg);
		}

		20% {
			transform: rotate(-10deg);
		}

		30% {
			transform: rotate(5deg);
		}

		40% {
			transform: rotate(-5deg);
		}

		50%,
		100% {
			transform: rotate(0deg);
		}
	}

	.treasureBox {
		animation: zy 1.5s 0.15s linear infinite;
	}
</style>