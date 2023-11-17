<template>
  <q-page padding>
    <div class="text-h5 q-mb-lg">User Level Settings</div>

    <q-form @submit="onSubmit">
      <div
        v-for="(field, index) in fields"
        :key="field.key"
        class="row items-center q-col-gutter-md q-mb-md"
      >
        <div class="col-auto q-pt-none" style="width: 96px">
          Level {{ index + 1 }}
        </div>

        <q-input
          class="col"
          :name="`level_${index + 1}_min_points`"
          type="number"
          v-model.number="field.value.min_points"
          clearable
          label="Min points"
          outlined
          disable
          bg-color="grey-1"
          :error="false"
        />

        <q-input
          class="col"
          :name="`level_${index + 1}_max_points`"
          type="number"
          :autofocus="field.isFirst"
          v-model.number="field.value.max_points"
          @update:model-value="(value) => editMaxPoints(index, value)"
          clearable
          label="Max points"
          outlined
          bg-color="grey-1"
          :error="Boolean(maxPointsErrors[index])"
          :error-message="maxPointsErrors[index]"
        />

        <q-input
          class="col"
          :name="`level_${index + 1}_discount`"
          type="number"
          v-model.number="field.value.discount"
          clearable
          outlined
          label-slot
          bg-color="grey-1"
          :error="Boolean(discountErrors[index])"
          :error-message="discountErrors[index]"
        >
          <template #label>
            <span>Discount </span>
            <span class="text-negative">*</span>
          </template>
        </q-input>
      </div>

      <div class="text-center">
        <q-btn type="submit" color="positive" label="Submit" />
      </div>
    </q-form>

    <pre>values: {{ values }}</pre>
    <pre>errors: {{ errors }}</pre>
  </q-page>
</template>

<script setup lang="ts">
import { computed } from 'vue';
import { useForm, useFieldArray } from 'vee-validate';
import { toTypedSchema } from '@vee-validate/yup';
import { object, number, array, ref } from 'yup';

const { values, errors, handleSubmit } = useForm({
  validationSchema: toTypedSchema(
    object({
      settings: array()
        .required()
        .min(1)
        .of(
          object({
            min_points: number().required().integer().min(0),
            max_points: number()
              .integer('Max points must be an integer')
              .min(
                ref('min_points'),
                'Max points must be greater than min points'
              )
              .transform((value, origin) =>
                origin != null && origin !== '' ? value : undefined
              ),
            discount: number()
              .required('Discount is required')
              .integer('Discount must be an integer')
              .min(0, 'Discount must be positive')
              .max(100, 'Discount must not exceed 100')
              .transform((value, origin) =>
                origin != null && origin !== '' ? value : undefined
              ),
          })
        )
        .test((value, ctx) => {
          for (let i = 1; i < value.length; ++i) {
            if (value[i - 1].max_points === undefined)
              return ctx.createError({
                message: 'Max points is required',
                path: `settings[${i - 1}].max_points`,
              });

            if (value[i].discount <= value[i - 1].discount)
              return ctx.createError({
                message: 'Discount should be higher than the previous one',
                path: `settings[${i}].discount`,
              });
          }

          return true;
        })
        .default([
          { min_points: 0, max_points: 0, discount: 0 },
          { min_points: 1, discount: 1 },
        ]),
    })
  ),
});

const { fields, update, push, remove } = useFieldArray<{
  min_points: number;
  max_points?: number;
  discount: number;
}>('settings');

const maxPointsErrors = computed(() =>
  fields.value.map(
    (_, index) =>
      errors.value[`settings[${index}].max_points` as never] ||
      errors.value[`settings[${index}]` as never]
  )
);

const discountErrors = computed(() =>
  fields.value.map(
    (_, index) => errors.value[`settings[${index}].discount` as never]
  )
);

function editMaxPoints(index: number, value: string | number | null) {
  if (fields.value[index].isLast) {
    push({
      min_points: Number(value) + 1,
      discount: fields.value[index].value.discount + 1,
    });
  } else if (
    index === fields.value.length - 2 &&
    (value == null || String(value) === '')
  ) {
    for (
      let i = index;
      i >= 0 &&
      (fields.value[i].value.max_points == null ||
        String(fields.value[i].value.max_points) === '');
      --i
    ) {
      remove(i + 1);
    }
  } else {
    update(index + 1, {
      min_points: Number(value) + 1,
      max_points: fields.value[index + 1].value.max_points,
      discount: fields.value[index + 1].value.discount,
    });
  }
}

const onSubmit = handleSubmit((values) => {
  console.log('Submitted with', values);
});
</script>
