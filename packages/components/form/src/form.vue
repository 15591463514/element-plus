<template>
  <form :class="formClasses">
    <slot />
  </form>
</template>

<script lang="ts" setup>
import { computed, provide, reactive, toRefs, watch } from 'vue'
import { debugWarn, isFunction } from '@element-plus/utils'
import { useNamespace } from '@element-plus/hooks'
import { useFormSize } from './hooks'
import { formContextKey } from './constants'
import { formEmits, formProps } from './form'
import { filterFields, useFormLabelWidth } from './utils'

import type { ValidateFieldsError } from 'async-validator'
import type { Arrayable } from '@element-plus/utils'
import type {
  FormContext,
  FormItemContext,
  FormValidateCallback,
  FormValidationResult,
} from './types'
import type { FormItemProp } from './form-item'

const COMPONENT_NAME = 'ElForm'
defineOptions({
  name: COMPONENT_NAME,
})
const props = defineProps(formProps)
const emit = defineEmits(formEmits)

/*
  表单字段值
    由form来统计, 抛出addField等方法, 来给父级的form添加表单字段
*/
const fields: FormItemContext[] = []

const formSize = useFormSize()
const ns = useNamespace('form')
const formClasses = computed(() => {
  const { labelPosition, inline } = props
  return [
    ns.b(),
    // todo: in v2.2.0, we can remove default
    // in fact, remove it doesn't affect the final style
    ns.m(formSize.value || 'default'),
    {
      [ns.m(`label-${labelPosition}`)]: labelPosition,
      [ns.m('inline')]: inline,
    },
  ]
})

const getField: FormContext['getField'] = (prop) => {
  return fields.find((field) => field.prop === prop)
}

const addField: FormContext['addField'] = (field) => {
  fields.push(field)
}

const removeField: FormContext['removeField'] = (field) => {
  if (field.prop) {
    fields.splice(fields.indexOf(field), 1)
  }
}

const resetFields: FormContext['resetFields'] = (properties = []) => {
  if (!props.model) {
    debugWarn(COMPONENT_NAME, 'model is required for resetFields to work.')
    return
  }
  filterFields(fields, properties).forEach((field) => field.resetField())
}

const clearValidate: FormContext['clearValidate'] = (props = []) => {
  filterFields(fields, props).forEach((field) => field.clearValidate())
}

const isValidatable = computed(() => {
  const hasModel = !!props.model
  if (!hasModel) {
    debugWarn(COMPONENT_NAME, 'model is required for validate to work.')
  }
  return hasModel
})

const obtainValidateFields = (props: Arrayable<FormItemProp>) => {
  if (fields.length === 0) return []

  const filteredFields = filterFields(fields, props)
  if (!filteredFields.length) {
    debugWarn(COMPONENT_NAME, 'please pass correct props!')
    return []
  }
  return filteredFields
}

/*
  ele的的form上的校验函数, 这个函数会校验所有有prop的表单项
  validateField第一个参数为空, 是校验所有表单属性
  直接返回了校验函数的执行结果
 */
const validate = async (
  callback?: FormValidateCallback
): FormValidationResult => validateField(undefined, callback)

/*
  异步验证表单字段
*/
const doValidateField = async (
  props: Arrayable<FormItemProp> = []
): Promise<boolean> => {
  // 字段不可验证
  if (!isValidatable.value) return false

  // 获取需要验证的字段
  const fields = obtainValidateFields(props)
  // 校验通过
  if (fields.length === 0) return true

  // 用于存储验证错误的变量
  let validationErrors: ValidateFieldsError = {}
  // 遍历fields数组, 对每个字段进行验证
  for (const field of fields) {
    try {
      await field.validate('')
    } catch (fields) {
      // 如果验证失败，则将错误信息存储到 validationErrors 对象中。
      validationErrors = {
        ...validationErrors,
        ...(fields as ValidateFieldsError),
      }
    }
  }

  // 如果存储错误的对象为空, 则说明验证成功, 返回true
  if (Object.keys(validationErrors).length === 0) return true
  // 否则, 验证失败, 返回验证错误信息
  return Promise.reject(validationErrors)
}

/*
  校验表单的函数, 这个函数会校验所有有prop的表单项
  如果校验失败, 会返回一个Promise.reject, 里面包含了校验失败的信息
  如果校验成功, 会返回一个Promise.resolve
*/
const validateField: FormContext['validateField'] = async (
  modelProps = [],
  callback
) => {
  /**
    函数检查callback是否是一个函数。如果不是，则将shouldThrow设置为true，表示在验证失败时应该抛出错误。
   */
  const shouldThrow = !isFunction(callback)
  try {
    // 校验所有的表单字段(modelProps是空数组)
    const result = await doValidateField(modelProps)
    // When result is false meaning that the fields are not validatable
    // 翻译: 当结果为假时，表示字段不可验证
    if (result === true) {
      await callback?.(result)
    }
    return result
  } catch (e) {
    if (e instanceof Error) throw e

    // 变量: 接受校验结果的错误对象
    const invalidFields = e as ValidateFieldsError

    if (props.scrollToError) {
      scrollToField(Object.keys(invalidFields)[0])
    }

    await callback?.(false, invalidFields)
    // 直接把错误对象返回
    return shouldThrow && Promise.reject(invalidFields)
  }
}

const scrollToField = (prop: FormItemProp) => {
  const field = filterFields(fields, prop)[0]
  if (field) {
    field.$el?.scrollIntoView(props.scrollIntoViewOptions)
  }
}

watch(
  () => props.rules,
  () => {
    if (props.validateOnRuleChange) {
      validate().catch((err) => debugWarn(err))
    }
  },
  { deep: true }
)

provide(
  formContextKey,
  reactive({
    ...toRefs(props),
    emit,

    resetFields,
    clearValidate,
    validateField,
    getField,
    addField,
    removeField,

    ...useFormLabelWidth(),
  })
)

defineExpose({
  /**
   * @description Validate the whole form. Receives a callback or returns `Promise`.
   */
  validate,
  /**
   * @description Validate specified fields.
   */
  validateField,
  /**
   * @description Reset specified fields and remove validation result.
   */
  resetFields,
  /**
   * @description Clear validation message for specified fields.
   */
  clearValidate,
  /**
   * @description Scroll to the specified fields.
   */
  scrollToField,
  /**
   * @description All fields context.
   */
  fields,
})
</script>
