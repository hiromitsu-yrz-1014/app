<template>
  <div class="min-h-screen flex items-center justify-center bg-gray-100 font-sans">
    <div class="max-w-md w-full bg-white p-8 rounded-lg shadow-md">
      <div class="flex justify-center mb-6">
        <h1 class="text-2xl font-bold text-blue-600">建設リソース管理</h1>
      </div>
      <h2 class="text-xl font-semibold mb-6 text-center text-gray-700">ログイン</h2>
      
      <form @submit.prevent="handleLogin" class="space-y-4">
        <div>
          <label class="block text-sm font-medium text-gray-700">メールアドレス</label>
          <input v-model="email" type="email" required class="mt-1 block w-full border border-gray-300 rounded-md shadow-sm p-2" placeholder="you@example.com" />
        </div>
        <div>
          <label class="block text-sm font-medium text-gray-700">パスワード</label>
          <input v-model="password" type="password" required class="mt-1 block w-full border border-gray-300 rounded-md shadow-sm p-2" />
        </div>
        
        <button type="submit" :disabled="loading" class="w-full bg-blue-600 text-white p-2 rounded-md hover:bg-blue-700 transition disabled:bg-blue-300">
          {{ loading ? 'ログイン中...' : 'ログイン' }}
        </button>
      </form>

      <p v-if="errorMsg" class="mt-4 text-red-500 text-sm text-center">{{ errorMsg }}</p>
    </div>
  </div>
</template>

<script setup>
const client = useSupabaseClient()
const email = ref('')
const password = ref('')
const loading = ref(false)
const errorMsg = ref('')

const handleLogin = async () => {
  try {
    loading.value = true
    errorMsg.value = ''
    
    // Supabaseの認証機能を使ってログイン
    const { error } = await client.auth.signInWithPassword({
      email: email.value,
      password: password.value,
    })
    
    if (error) throw error
    
    // 成功したらトップページ（/）へ移動
    alert('ログインに成功しました！')
    navigateTo('/')
  } catch (error) {
    errorMsg.value = 'ログインに失敗しました: ' + error.message
  } finally {
    loading.value = false
  }
}
</script>