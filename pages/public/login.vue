<template>
	<view class="container">
		<view class="mix-list-cell b-b">
			<view class="region">
				<text>+86</text>
				<text class="yticon icon-xia"></text>
			</view>
			<input type="number" maxlength="11" class="cell-content" v-model="mobile" placeholder="输入手机号(新号码自动注册)" />
		</view>
		<view class="mix-list-cell b-b">
			<input type="number" maxlength="6" class="cell-content" v-model="code" placeholder="输入验证码" />
			<button class="add-btn" :class="{ disabled: disabled || processing }" @click="getCode()">{{ btnGetCodeText }}</button>
		</view>
		<view class="mix-list-cell-buttons"><button class="add-btn" :class="{ disabled: disabledLogin }" @click="btnLogin()">登录</button></view>
		<!-- #ifdef APP-PLUS-->
		<view class="mix-list-cell-buttons"><button class="add-btn" @click="jgLogin()">本机号码一键登录</button></view>
		<!-- #endif-->

		<view class="bottom">
			<text>登录即代表你同意</text>
			<text class="link" @click="navToDocPage('1229caae5ef574ae0055cfa40630f5c5')">《服务协议》</text>
			<text>和</text>
			<text class="link" @click="navToDocPage('a3e75f055ef574c200492faa50335f39')">《隐私协议》</text>
		</view>
	</view>
</template>

<script>
//#ifdef APP-PLUS
//极光登录插件
const jv = uni.requireNativePlugin('JG-JVerification');
//#endif
import { mobileAutoLogin, getToken, mobileLogin } from '@/common/request.js';
import { mapMutations } from 'vuex';
import { client, auth } from '@/common/cloud.js';
import { navToDocPage } from '@/common/functions.js';
export default {
	data() {
		return {
			platform: '',
			loadingText: '安全检测中',
			loginErrorText: '检测失败!\n请打开4G网络后再尝试!',
			ios_uiConfigure: {},
			logining: false,
			mobile: '',
			mobileReg: /^1[3456789]\d{9}$/,
			disabled: true,
			processing: false,
			disabledLogin: true,
			btnGetCodeOldText: '获取验证码',
			btnGetCodeText: '获取验证码',
			daojishiTotal: 60,
			code: ''
		};
	},
	watch: {
		mobile(newText, oldText) {
			if (this.mobileReg.test(newText)) {
				this.disabled = false;
			} else {
				this.disabled = true;
			}
		},
		code(newText, oldText) {
			if (/\d{6}$/.test(newText) && !this.disabled) {
				this.disabledLogin = false;
			} else {
				this.disabledLogin = true;
			}
		}
	},
	onLoad() {
		/* this.platform = uni.getSystemInfoSync().platform;
		//测试阶段，自动登录
		if(this.platform  == "ios"){
			this.toLogin();
			return;
		} */
		//在应用初始化调用
		//#ifdef APP-PLUS
		this.jgLogin();
		//#endif

		//#ifdef H5
		//this.toLogin();
		//#endif
		//this.getPhoneInfo();
	},
	methods: {
		...mapMutations(['login']),

		async jgLogin() {
			uni.showLoading({
				mask: true,
				title: this.loadingText
			});
			if (jv) {
				jv.checkVerifyEnable(result => {
					if (result.enable) {
						this.loginAuth();
					} else {
						//this.$api.msg(this.loginErrorText,2000);
						//this.navTimeBack();
						//页面输入手机号+验证码登录
						uni.hideLoading();
					}
				});
			} else {
				uni.hideLoading();
			}
		},
		loginAuth() {
			uni.hideLoading();
			jv.loginAuth(
				{
					autoFinish: true,
					timeout: 5000
				},
				result => {
					// 结果监听
					console.log('jv.loginAuth', result);
					if (result.code == 6000) {
						//登录成功
						this.loginWithToken(result.content);
					} else {
						//手动返回
						this.navBack();
					}
				},
				event => {
					// 事件监听
					let code = event.code;
					let eventDesc = event.content;
					console.log('jv.loginAuth', event);
				}
			);
		},
		loginWithToken(token) {
			mobileAutoLogin({
				token: token
			}).then(
				res => {
					console.log(res);
					//this.login(res);
					this.loginSuccess(res);
				},
				err => {
					this.$api.msg('登录失败' + err.message);
					this.navTimeBack();
				}
			);
		},
		navTimeBack() {
			setTimeout(() => {
				this.navBack();
			}, 2000);
		},
		navBack() {
			uni.navigateBack();
		},
		navToDocPage(id) {
			navToDocPage(id);
		},
		async toLogin() {
			this.logining = true;
			//const { mobile, password } = this;
			const result = await this.$api.json('userInfo');
			if (result.status === 1) {
				this.$api.msg('登录成功', 2000, false, 'success');
				//this.login(result.data);
				getToken({
					uid: result.data.id,
					token: result.data.token
				}).then(
					res => {
						console.log(res);
						//写入缓存
						this.loginSuccess(res);
					},
					err => {
						//登录信息失效
						uni.removeStorage({
							key: 'userInfo'
						});
					}
				);
				setTimeout(() => {
					uni.navigateBack();
				}, 2000);
			} else {
				this.$api.msg(result.msg);
				this.logining = false;
			}
		},
		getCode() {
			if (this.disabled) {
				return;
			}
			//再次验证手机号码
			if (!this.checkMobile()) {
				this.$api.msg('手机号格式不正确');
				return;
			}
			//发送短信,但是需要企业信息,这里做假的功能,假如调用云函数，发送短信成功....
			//this.$api.msg("发送短信成功！",3000);
			this.$api.msg('测试例子，任意验证码都能通过', 3000);
			//倒计时60秒
			this.daojishi(this.daojishiTotal);
		},
		daojishi(sec) {
			if (sec > 0) {
				sec--;
				this.btnGetCodeText = sec + 's';
				this.processing = true;
				setTimeout(() => {
					this.daojishi(sec);
				}, 1000);
			} else {
				this.processing = false;
				this.btnGetCodeText = this.btnGetCodeOldText;
			}
		},
		btnLogin() {
			if (this.disabled || this.disabledLogin) {
				return;
			}
			//再次验证手机号码
			if (!this.checkMobile()) {
				this.$api.msg('手机号格式不正确');
				return;
			}
			//登录，手机号+验证码
			mobileLogin({
				mobile: this.mobile,
				code: this.code
			}).then(
				res => {
					console.log(res);
					this.loginSuccess(res);
				},
				err => {
					this.$api.msg('登录失败');
				}
			);
		},
		checkMobile() {
			return this.mobileReg.test(this.mobile);
		},
		loginSuccess(res) {
			this.$api.msg('登录成功', 2000, false, 'success');
			//写入缓存
			uni.setStorage({
				key: 'userInfo',
				data: res
			});
			auth.signInWithTicket(res.ticket).then(() => {
				// 登录成功
				console.log('客户端登录成功');
			});
			this.navTimeBack();
		}
	}
};
</script>

<style lang="scss">
page {
}
.mix-list-cell {
	display: flex;
	align-items: baseline;
	padding: 20upx $page-row-spacing;
	line-height: 60upx;
	position: relative;

	&.cell-hover {
		background: #fafafa;
	}
	&.b-b:after {
		left: 30upx;
	}

	.cell-icon {
		align-self: center;
		width: 56upx;
		max-height: 60upx;
		font-size: 38upx;
	}
	.cell-more {
		align-self: center;
		font-size: 30upx;
		color: $font-color-base;
		margin-left: $uni-spacing-row-sm;
	}
	.cell-tit {
		font-size: $font-base;
		color: $font-color-dark;
		margin-right: 20upx;
	}
	.cell-content {
		flex: 1;
		font-size: $font-sm + 2upx;
	}
	.cell-tip {
		flex: 1;
		font-size: $font-sm + 2upx;
		color: $font-color-light;
		text-align: right;
	}
}
.region {
	font-size: $font-base;
	margin-right: 8upx;
	display: flex;
	align-items: center;
}
.add-btn {
	display: flex;
	align-items: center;
	justify-content: center;
	font-size: $font-base;
	color: #fff;
	background-color: $btn-color-light;
	border-radius: 10upx;
	&.disabled {
		background-color: $font-color-disabled;
	}
}
.mix-list-cell-buttons {
	padding: 40upx;
}
.bottom {
	font-size: $font-sm;
	text-align: center;
	position: absolute;
	bottom: 40upx;
	width: 100%;
}
.link {
	color: $base-color;
}
</style>
