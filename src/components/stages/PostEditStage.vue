<template>
  <section v-if="post">
    <b-field
      v-if="post.author"
      label="Author">
      <b-input
        disabled
        v-model="post.author.username"/>
    </b-field>

    <b-field label="Heading">
      <b-input v-model="post.heading"/>
    </b-field>

    <b-field label="Topics">
      <b-taginput
        autocomplete
        v-model="post.topics"
        :data="filteredTopics"
        icon="label"
        :allow-new="true"
        placeholder="Add a topic"
        :rounded="true"
        @typing="getFilteredTopics"/>
    </b-field>

    <b-field label="Message">
      <wysiwyg-editor
        :message="post.message"
        ref="editorMessage"/>

    </b-field>
    <p class="level">
      <button
        class="button save-btn"
        @click="editPost(post.heading, ($refs.editorMessage.editor.getValue()), post.topics, post.promoted)">Save changes</button>

      <b-switch v-model="post.promoted">
        Promoted
      </b-switch>
    </p>
    <nav class="level">
      <div class="level-item has-text-centered"/>
      <b-message
        v-if="saveStatus === 'saved'"
        class="level-item"
        type="is-success"
        has-icon>
        Changes were saved!
        <router-link :to="`/post/${this.$route.params.id}`">Go to post</router-link>
      </b-message>
      <b-message
        v-if="saveStatus === 'nochange'"
        class="level-item"
        type="is-warning"
        has-icon>
        No changes were made
      </b-message>
      <div class="level-item has-text-centered"/>
    </nav>
  </section>
</template>

<script>
import WysiwygEditor from './WysiwygEditor'
import { storage } from '../../main.js'
const youtubeRegex = /(?:http:|https:)?\/\/(?:www\.)?(?:youtube.com|youtu.be)\/(?:watch)(?:\?v=)?(?:\S{11})(]|)/g

export default {
  components: {
    WysiwygEditor
  },
  props: ['post'],
  data () {
    return {
      filteredTopics: [],
      isTrue: true,
      initialValues: {},
      saveStatus: false
    }
  },
  computed: {
    topicsArray: function () {
      return Object.keys(this.topics)
    }
  },
  watch: {
    post: function () {
      this.$refs.editorMessage.editor.setValue('')
      this.$refs.editorMessage.editor.insertText(this.post.message)
      this.setInitialValues()
    }
  },
  mounted () {
    this.setInitialValues()
    this.$store.dispatch('getTopics')
  },
  destroyed () {
    this.$store.dispatch('clearImageCache')
  },
  methods: {
    getFilteredTopics (text) {
     this.filteredTopics = [text, ...this.$store.getters.filteredTopics(text).map((topic) => { return topic.topic })]
    },
    setInitialValues () {
      if (Object.keys(this.post).length === 0) return
      this.initialValues = Object.assign({}, this.post)
      this.initialValues = Object.assign(this.initialValues, { message: this.$refs.editorMessage.editor.getValue(), topics: this.post.topics.slice() })
    },
    editPost (heading, message, topics, promoted) { // <-- and here
      const editedAt = new Date()
      let payload = {heading, message, topics, promoted}

      for (let attr in payload) {
        if (payload[attr] === this.initialValues[attr]) {
          delete payload[attr]
        }
      }

      if (Object.keys(payload).length > 0) {
        for (let image in this.imageCache) {
          if (!this.imageCache[image].storagePath) {
            console.log('All images not yet uploaded PLACEHOLDER')
            return
          }
          payload.message = payload.message.replace(this.imageCache[image].blobPath, this.imageCache[image].storagePath)
        }
        if (youtubeRegex.test(payload.message)) {
          payload.message = this.replaceYouTubeUrls(payload.message)
        }
        Object.assign(payload, { editedAt, id: this.$route.params.id })
        this.$store.dispatch('editPost', payload)
        this.saveStatus = 'saved'
      } else {
        this.saveStatus = 'nochange'
      }
    },
    replaceYouTubeUrls (message) {
      let width = 560
      let height = 315
      let matchedUrls = message.match(youtubeRegex)

      matchedUrls.forEach((url) => {
        let replaceWith = '';

        if(url.endsWith(']')){
          url = '[' + url
        }else{
        let videoId = this.getYouTubeId(url)
        replaceWith = '<p style="text-align: center;"><iframe width="' + width + '" height="' + height + '" src="//www.youtube.com/embed/' +
        videoId + '" frameborder="0" allowfullscreen></iframe></p>'
        }

        message = message.replace('('+url+')', replaceWith)
        message = message.replace(url, replaceWith)
      })
      return message
    },
    getYouTubeId (url) {
      var regExp = /^.*(youtu.be\/|v\/|u\/\w\/|embed\/|watch\?v=|\&v=)([^#\&\?]*).*/
      var match = url.match(regExp)

      if (match && match[2].length == 11) {
        return match[2]
      } else {
        return 'error'
      }
    }
  }
}
</script>

<style scoped lang="scss">
@import "../../styles/variables.scss";
.box {
  background-color: #fff;
}

.message{
  max-width:400px;
}

.save-btn {
  @include btn;
}
</style>
