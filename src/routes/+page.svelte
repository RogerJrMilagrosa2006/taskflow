<script lang="ts">
  import { supabase } from '../lib/supabaseClient';

  let mode = $state<'login' | 'signup'>('login');
  let email = $state('');
  let password = $state('');
  let loading = $state(false);
  let message = $state('');
  let errorMessage = $state('');

  async function handleSubmit(event: SubmitEvent) {
    event.preventDefault();

    message = '';
    errorMessage = '';
    loading = true;

    try {
      if (mode === 'login') {
        const { error } = await supabase.auth.signInWithPassword({
          email: email.trim(),
          password
        });

        if (error) throw error;

        window.location.href = '/dashboard';
      } else {
        const { data, error } = await supabase.auth.signUp({
          email: email.trim(),
          password
        });

        if (error) throw error;

        if (data.session) {
          window.location.href = '/dashboard';
        } else {
          message =
            'Account created! Check your email to confirm your account before logging in.';
        }
      }
    } catch (error) {
      errorMessage =
        error instanceof Error ? error.message : 'Something went wrong.';
    } finally {
      loading = false;
    }
  }
</script>

<svelte:head>
  <title>{mode === 'login' ? 'Login' : 'Sign Up'} | Taskflow</title>
  <meta
    name="description"
    content="Sign in to your private Taskflow Todo List."
  />
</svelte:head>

<main class="page">
  <section class="card">
    <div class="logo">✓</div>
    <h1>Taskflow</h1>

    <p class="subtitle">
      {mode === 'login'
        ? 'Welcome back! Sign in to your account.'
        : 'Create an account for your personal task list.'}
    </p>

    <form onsubmit={handleSubmit}>
      <label for="email">Email address</label>
      <input
        id="email"
        type="email"
        bind:value={email}
        placeholder="you@example.com"
        autocomplete="email"
        required
      />

      <label for="password">Password</label>
      <input
        id="password"
        type="password"
        bind:value={password}
        placeholder="Enter your password"
        autocomplete={mode === 'login' ? 'current-password' : 'new-password'}
        minlength="6"
        required
      />

      {#if errorMessage}
        <p class="error" role="alert">{errorMessage}</p>
      {/if}

      {#if message}
        <p class="success" role="status">{message}</p>
      {/if}

      <button type="submit" disabled={loading}>
        {loading
          ? 'Please wait...'
          : mode === 'login'
            ? 'Log In'
            : 'Create Account'}
      </button>
    </form>

    <p class="switch">
      {mode === 'login'
        ? "Don't have an account?"
        : 'Already have an account?'}

      <button
        type="button"
        class="link"
        onclick={() => {
          mode = mode === 'login' ? 'signup' : 'login';
          message = '';
          errorMessage = '';
        }}
      >
        {mode === 'login' ? 'Sign Up' : 'Log In'}
      </button>
    </p>
  </section>
</main>

<style>
  :global(body) {
    margin: 0;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    background: #f5f6fb;
    color: #202338;
  }

  .page {
    min-height: 100vh;
    display: grid;
    place-items: center;
    padding: 24px;
    box-sizing: border-box;
  }

  .card {
    width: 100%;
    max-width: 400px;
    box-sizing: border-box;
    padding: 36px;
    border-radius: 20px;
    background: white;
    box-shadow: 0 12px 40px #20233812;
  }

  .logo {
    width: 48px;
    height: 48px;
    display: grid;
    place-items: center;
    margin: 0 auto 16px;
    border-radius: 14px;
    background: #6658d3;
    color: white;
    font-size: 27px;
    font-weight: 700;
  }

  h1 {
    margin: 0;
    text-align: center;
    font-size: 28px;
  }

  .subtitle {
    margin: 10px 0 28px;
    text-align: center;
    color: #777b8d;
    font-size: 14px;
    line-height: 1.6;
  }

  form {
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  label {
    margin-top: 8px;
    font-size: 13px;
    font-weight: 600;
  }

  input {
    width: 100%;
    box-sizing: border-box;
    padding: 13px 14px;
    border: 1px solid #dfe1eb;
    border-radius: 10px;
    font: inherit;
    outline-color: #6658d3;
  }

  button[type="submit"] {
    margin-top: 14px;
    padding: 14px;
    border: 0;
    border-radius: 10px;
    background: #6658d3;
    color: white;
    font: inherit;
    font-weight: 600;
    cursor: pointer;
  }

  button:disabled {
    opacity: 0.65;
    cursor: wait;
  }

  .switch {
    margin: 22px 0 0;
    text-align: center;
    color: #777b8d;
    font-size: 13px;
  }

  .link {
    border: 0;
    background: none;
    color: #6658d3;
    font: inherit;
    font-weight: 600;
    cursor: pointer;
  }

  .error,
  .success {
    font-size: 13px;
    line-height: 1.5;
    overflow-wrap: anywhere;
  }

  .error {
    color: #c0392b;
  }

  .success {
    color: #217a4b;
  }
</style>