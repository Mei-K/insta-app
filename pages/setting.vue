<template>
  <div class="container mx-auto mt-6">
    <!-- icon -->
    <div class="grid justify-items-center py-6">
      <img class="h-28 w-28 rounded-full object-cover" :src="icon" />
      <span class="pt-2 text-sm text-warning-color h-6">
        {{ errorIcon }}
      </span>
      <label for="file-upload" class="cursor-pointer rounded-md px-4">
        <span class="text-xl text-accent-color">
          Change Profile Photo
          <input
            id="file-upload"
            name="file-upload"
            type="file"
            class="sr-only"
            accept="image/jpeg, image/jpg, image/png"
            @change="fileSelected"
        /></span>
      </label>
    </div>
    <!-- input -->
    <div class="grid grid-cols-4 gap-2 border-t border-light-gray px-4 py-8">
      <div class="col-span-1 py-2">User Name</div>
      <div
        class="col-span-3 border-b border-light-gray text-input-value-color py-2 pl-2"
      >
        <input
          v-model="userName"
          type="text"
          class="appearance-none border-none w-full focus:outline-none"
        />
      </div>
      <div class="col-start-2 col-span-3 text-sm pl-2 h-6 text-warning-color">
        {{ errorUserName }}
      </div>
      <div class="col-span-1 py-2">Bio</div>
      <div
        class="col-span-3 border-b border-light-gray text-input-value-color py-2 pl-2"
      >
        <input
          v-model="bio"
          type="text"
          class="appearance-none border-none w-full focus:outline-none"
        />
      </div>
    </div>
    <!-- button -->
    <div class="px-4 pb-4 text-center mt-2">
      <button
        type="button"
        class="inline-flex justify-center py-2 px-4 text-xl text-dark-gray"
        @click="cancel"
      >
        Cancel
      </button>
      <button
        class="inline-flex justify-center py-2 px-4 text-xl text-accent-color"
        type="button"
        @click="updateUserInfo"
      >
        Done
      </button>
    </div>
    <!-- logout -->
    <div class="grid justify-items-center border-t border-light-gray py-8">
      <button
        class="inline-flex justify-center py-2 px-4 text-xl text-warning-color"
        @click="logout"
      >
        Log out
      </button>
    </div>
  </div>
</template>

<script lang="ts">
import Vue from 'vue'
import Auth from '../plugins/auth'

export default Vue.extend({
  middleware: 'auth',
  data() {
    return {
      // アイコンのURL
      icon: '',
      // ユーザー名
      userName: '',
      // 自己紹介文
      bio: '',
      // ユーザー名エラー文
      errorUserName: '',
      // アイコンエラー文
      errorIcon: '',
      // S3に送信する画像ファイル
      iconFile: {},
      // アップデート用アイコンURL
      imageUrl: '',
      // S3に送信する削除するアイコンURL
      deleteIcon: '',
    }
  },
  /**
   * ログインしているユーザーのアイコン、ユーザー名、自己紹介を取得する.
   */
  created() {
    this.showUserInfo()
  },
  methods: {
    /**
     * 現在のユーザー情報を表示する.
     */
    async showUserInfo() {
      const res = await this.$store.getters['user/getLoginUserInfo']
      if (res.icon === '') {
        this.icon = '/images/user.png'
      } else {
        this.icon = res.icon
        this.imageUrl = res.icon
        this.deleteIcon = res.icon
      }
      this.userName = res.userName
      this.bio = res.bio
    },
    /**
     * 新しいアイコン画像に入れ替える.
     * @param e - 添付ファイル
     */
    fileSelected(e: any): void {
      this.errorIcon = ''
      const file = e.target.files[0]
      if (file) {
        // 制限サイズ(3MB)
        const sizeLimit = 1024 * 1024 * 3
        // ファイルサイズが制限以上の場合のエラー
        if (file.size > sizeLimit) {
          this.errorIcon = 'ファイルサイズは3MB以下にしてください'
          return
        }

        this.icon = window.URL.createObjectURL(file)
        this.iconFile = file
      }
    },
    /**
     * アイコン、名前、自己紹介文を変更する.
     */
    async updateUserInfo() {
      // ユーザー名空欄チェック
      if (this.userName.length === 0) {
        this.errorUserName = 'ユーザー名を入力してください'
        return
      }
      // ローディング開始
      this.$nuxt.$loading.start()
      // アイコンの変更チェック
      if (this.icon !== this.imageUrl) {
        // S3からURLを取得
        const { url } = await fetch(
          'https://api-instagram-app.herokuapp.com/s3Url'
        ).then((res) => res.json())
        // S3のバケットに写真をPOST
        await fetch(url, {
          method: 'PUT',
          headers: {
            'Content-Type': 'multipart/form-data',
          },
          body: this.iconFile as any,
        })
        this.imageUrl = url.split('?')[0]
        // S3のバケットから写真を削除
        await this.$axios.delete(
          'https://api-instagram-app.herokuapp.com/s3Url',
          {
            data: { urlArray: [this.deleteIcon] },
          }
        )
      }
      // ログインしているユーザーIDを取得.
      const userId = await this.$store.getters['user/getLoginUserId']
      // APIに更新後のユーザー情報をPOST
      const res = await this.$axios.post(
        'https://api-instagram-app.herokuapp.com/setting',
        {
          userId,
          icon: this.imageUrl,
          userName: this.userName,
          bio: this.bio,
        }
      )
      // ローディング終了
      this.$nuxt.$loading.finish()
      // 変更成功時
      if (res.data.status === 'success') {
        // ユーザー情報をVuexに保管
        await this.$store.commit('user/setLoginUserInfo', res.data.data)
        await this.showUserInfo()
        // 変更失敗時
      } else if (res.data.status === 'error') {
        this.errorUserName = 'そのユーザー名は既に使われています'
      }
    },
    /**
     * 変更をキャンセルしマイページ画面に遷移する.
     */
    cancel() {
      this.$router.push('/Mypage')
    },
    /**
     * ログアウトしてログイン画面に遷移する.
     */
    logout() {
      // AuthプラグインでtokenをCookieから削除
      Auth.logout(this.$cookies)
      this.$router.push('/')
    },
  },
})
</script>
<style></style>
