<template>
	<view class="u-form-item">
		<view>{{label}}</view>
		<view class="">
			<slot />
			<view :class="isRequired?'ai-form-item-label-required':''" class="ai-form-item-message" v-if="validateState==='error'">{{validateMessage}}</view>
		</view>
	</view>
</template>

<script>
import Emitter from '../../libs/util/emitter.js';
import schema from '../../libs/util/async-validator';
export default {
	name: 'u-form-item',
	mixins: [Emitter],
	inject: ['uForm'],
	props: {
		// input的label提示语
		label: {
			type: String,
			default: ''
		},
		// 绑定的值
		prop: {
			type: String,
			default: ''
		}
	},
	data() {
		return {
			initialValue: '', // 存储的默认值
			isRequired: false, // 是否必填
			validateState: '', // 是否校验成功
			validateMessage: '' // 校验失败的提示语
		};
	},
	computed: {
		fieldValue() {
			return this.uForm.model[this.prop];
		}
	},
	methods: {
		// 判断是否需要required校验
		setRules() {
			let that = this;
			// 从父组件u-form拿到当前u-form-item需要验证 的规则
			let rules = this.getRules();
			if (rules.length) {
				this.isRequired = rules.some(rule => {
					// 如果有必填项，就返回，没有的话，就是undefined
					return rule.required;
				});
			}

			// blur事件
			this.$on('on-form-blur', that.onFieldBlur);
			// change事件
			this.$on('on-form-change', that.onFieldChange);
		},

		// 从u-form的rules属性中，取出当前u-form-item的校验规则
		getRules() {
			// 父组件的所有规则
			let rules = this.uForm.rules;
			rules = rules ? rules[this.prop] : [];
			// 保证返回的是一个数组形式
			return [].concat(rules || []);
		},

		// blur事件时进行表单校验
		onFieldBlur() {
			this.validation('blur');
		},

		// change事件进行表单校验
		onFieldChange() {
			this.validation('change');
		},

		// 过滤出符合要求的rule规则
		getFilteredRule(trigger) {
			let rules = this.getRules();
			// 历遍判断规则是否有对应的事件，比如blur，change触发等的事件
			return rules.filter(res => !res.trigger || trigger.indexOf(trigger) !== -1);
		},

		// 校验数据
		validation(trigger, callback = () => {}) {
			// blur和change是否有当前方式的校验规则
			let rules = this.getFilteredRule(trigger);
			// 判断是否有验证规则
			if (!rules || rules.length === 0) return;
			// 设置当前的装填，标识为校验中
			this.validateState = 'validating';
			// 调用async-validator的方法
			let validator = new schema({ [this.prop]: rules });
			validator.validate({ [this.prop]: this.fieldValue }, { firstFields: true }, (errors, fields) => {
				// 记录状态和报错信息
				this.validateState = !errors ? 'success' : 'error';
				this.validateMessage = errors ? errors[0].message : '';
				// 调用回调方法
				callback(this.validateMessage);
			});
		},

		// 清空当前的u-form-item
		resetField() {
			this.uForm.model[this.prop] = this.initialValue;
		}
	},

	// 组件创建完成时，将当前实例保存到u-form中
	mounted() {
		// 如果没有传入prop，就不进行校验
		if (!this.prop) return;
		// 发出事件，让父组件将本实例加入到管理数组中
		this.dispatch('u-form', 'on-form-item-add', this);
		// 设置初始值
		this.initialValue = this.fieldValue;
		// 添加表单校验
		this.setRules();
	},

	// 组件销毁前，将实例从 Form 的缓存中移除
	beforeDestroy() {
		this.dispatch('u-form', 'on-form-item-remove', this);
	}
};
</script>

<style></style>
