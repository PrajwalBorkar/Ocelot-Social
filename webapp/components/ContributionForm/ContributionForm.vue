<template>
  <ds-form
    class="contribution-form"
    ref="contributionForm"
    v-model="formData"
    :schema="formSchema"
    @submit="submit"
  >
    <template #default="{ errors }">
      <base-card>
        <template #heroImage>
          <img
            v-if="formData.image"
            :src="formData.image | proxyApiUrl"
            :class="['image', formData.imageBlurred && '--blur-image']"
          />
          <image-uploader
            :hasImage="!!formData.image"
            :class="[formData.imageBlurred && '--blur-image']"
            @addHeroImage="addHeroImage"
            @addImageAspectRatio="addImageAspectRatio"
            @addImageType="addImageType"
          />
        </template>
        <div v-if="formData.image" class="blur-toggle">
          <label for="blur-img">{{ $t('contribution.inappropriatePicture') }}</label>
          <input type="checkbox" id="blur-img" v-model="formData.imageBlurred" />
          <page-params-link class="link" :pageParams="links.FAQ">
            {{ $t('contribution.inappropriatePicture') }}
            <base-icon name="question-circle" />
          </page-params-link>
        </div>
        <ds-input
          model="title"
          :placeholder="$t('contribution.title')"
          name="title"
          autofocus
          size="large"
        />
        <ds-chip size="base" :color="errors && errors.title && 'danger'">
          {{ formData.title.length }}/{{ formSchema.title.max }}
          <base-icon v-if="errors && errors.title" name="warning" />
        </ds-chip>
        <hc-editor
          :users="users"
          :value="formData.content"
          :hashtags="hashtags"
          @input="updateEditorContent"
        />
        <ds-chip size="base" :color="errors && errors.content && 'danger'">
          {{ contentLength }}
          <base-icon v-if="errors && errors.content" name="warning" />
        </ds-chip>
        <categories-select
          v-if="categoriesActive"
          model="categoryIds"
          :existingCategoryIds="formData.categoryIds"
        />
        <ds-chip
          v-if="categoriesActive"
          size="base"
          :color="errors && errors.categoryIds && 'danger'"
        >
          {{ formData.categoryIds.length }} / 3
          <base-icon v-if="errors && errors.categoryIds" name="warning" />
        </ds-chip>
        <div class="buttons">
          <base-button data-test="cancel-button" :disabled="loading" @click="$router.back()" danger>
            {{ $t('actions.cancel') }}
          </base-button>
          <base-button type="submit" icon="check" :loading="loading" :disabled="errors" filled>
            {{ $t('actions.save') }}
          </base-button>
        </div>
      </base-card>
    </template>
  </ds-form>
</template>

<script>
import gql from 'graphql-tag'
import { mapGetters } from 'vuex'
import HcEditor from '~/components/Editor/Editor'
import PostMutations from '~/graphql/PostMutations.js'
import CategoriesSelect from '~/components/CategoriesSelect/CategoriesSelect'
import ImageUploader from '~/components/ImageUploader/ImageUploader'
import links from '~/constants/links.js'
import PageParamsLink from '~/components/_new/features/PageParamsLink/PageParamsLink.vue'

export default {
  components: {
    HcEditor,
    ImageUploader,
    PageParamsLink,
    CategoriesSelect,
  },
  props: {
    contribution: {
      type: Object,
      default: () => ({}),
    },
  },
  data() {
    const { title, content, image, categories } = this.contribution
    const {
      sensitive: imageBlurred = false,
      aspectRatio: imageAspectRatio = null,
      type: imageType = null,
    } = image || {}

    return {
      categoriesActive: this.$env.CATEGORIES_ACTIVE,
      links,
      formData: {
        title: title || '',
        content: content || '',
        image: image || null,
        imageAspectRatio,
        imageType,
        imageBlurred,
        categoryIds: categories ? categories.map((category) => category.id) : [],
      },
      formSchema: {
        title: { required: true, min: 3, max: 100 },
        content: { required: true },
        imageBlurred: { required: false },
        categoryIds: {
          type: 'array',
          required: this.categoriesActive,
          validator: (_, value = []) => {
            if (this.categoriesActive && (value.length === 0 || value.length > 3)) {
              return [new Error(this.$t('common.validations.categories'))]
            }
            return []
          },
        },
      },
      loading: false,
      users: [],
      hashtags: [],
      imageUpload: null,
    }
  },
  computed: {
    ...mapGetters({
      currentUser: 'auth/user',
    }),
    contentLength() {
      return this.$filters.removeHtml(this.formData.content).length
    },
  },
  methods: {
    submit() {
      let image = null
      const { title, content, categoryIds } = this.formData
      if (this.formData.image) {
        image = {
          sensitive: this.formData.imageBlurred,
        }
        if (this.imageUpload) {
          image.upload = this.imageUpload
          image.aspectRatio = this.formData.imageAspectRatio
          image.type = this.formData.imageType
        }
      }
      this.loading = true
      this.$apollo
        .mutate({
          mutation: this.contribution.id ? PostMutations().UpdatePost : PostMutations().CreatePost,
          variables: {
            title,
            content,
            categoryIds,
            id: this.contribution.id || null,
            image,
          },
        })
        .then(({ data }) => {
          this.loading = false
          this.$toast.success(this.$t('contribution.success'))
          const result = data[this.contribution.id ? 'UpdatePost' : 'CreatePost']

          this.$router.push({
            name: 'post-id-slug',
            params: { id: result.id, slug: result.slug },
          })
        })
        .catch((err) => {
          this.$toast.error(err.message)
          this.loading = false
        })
    },
    updateEditorContent(value) {
      this.$refs.contributionForm.update('content', value)
    },
    addHeroImage(file) {
      this.formData.image = null
      if (file) {
        const reader = new FileReader()
        reader.onload = ({ target }) => {
          this.formData.image = {
            ...this.formData.image,
            url: target.result,
          }
        }
        reader.readAsDataURL(file)
        this.imageUpload = file
      }
    },
    addImageAspectRatio(aspectRatio) {
      this.formData.imageAspectRatio = aspectRatio
    },
    addImageType(imageType) {
      this.formData.imageType = imageType
    },
  },
  apollo: {
    User: {
      query() {
        return gql`
          query {
            User(orderBy: slug_asc) {
              id
              slug
            }
          }
        `
      },
      result({ data: { User } }) {
        this.users = User
      },
    },
    Tag: {
      query() {
        return gql`
          query {
            Tag(orderBy: id_asc) {
              id
            }
          }
        `
      },
      result({ data: { Tag } }) {
        this.hashtags = Tag
      },
    },
  },
}
</script>

<style lang="scss">
.contribution-form > .base-card {
  display: flex;
  flex-direction: column;

  > .hero-image {
    position: relative;

    > .image {
      max-height: $size-image-max-height;
    }
  }

  .image.--blur-image {
    filter: blur($blur-radius);
  }

  > .ds-form-item {
    margin: 0;
  }

  > .ds-chip {
    align-self: flex-end;
    margin: $space-xx-small 0 $space-base;
    cursor: default;
  }

  > .select-field {
    align-self: flex-end;
  }

  > .buttons {
    align-self: flex-end;
    margin-top: $space-base;
  }

  .blur-toggle {
    text-align: right;
    margin-bottom: $space-base;

    > .link {
      display: block;
    }
  }
}
</style>
