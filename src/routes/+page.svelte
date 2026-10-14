
<script lang="ts">
  import { onMount } from 'svelte';
  import { supabase } from '../lib/supabaseClient';

  type Todo = {
    id: string | number;
    title: string;
    description: string;
    completed: boolean;
    createdAt: string;
    updatedAt?: string;
  };

  type TodoRow = {
    id: number;
    title: string;
    description: string | null;
    completed: boolean;
    created_at: string;
    updated_at: string | null;
  };

  let todos = $state<Todo[]>([]);
  let title = $state('');
  let description = $state('');
  let filter = $state('all');
  let editingId = $state<string | number | null>(null);
  let editTitle = $state('');
  let editDescription = $state('');
  let storageReady = $state(false);
  let storageError = $state(false);

  function fromDatabase(row: TodoRow): Todo {
    return {
      id: row.id,
      title: row.title,
      description: row.description ?? '',
      completed: row.completed,
      createdAt: row.created_at,
      updatedAt: row.updated_at ?? undefined
    };
  }

  async function getUserId(): Promise<string> {
    const { data, error } = await supabase.auth.getUser();

    if (error) throw error;
    if (!data.user) throw new Error('Please log in to manage your tasks.');

    return data.user.id;
  }

  
async function loadTodos() {
  storageReady = false;
  storageError = false;

  try {
    const userId = await getUserId();

    const { data, error } = await supabase
      .from('todos')
      .select('id, title, description, completed, created_at, updated_at')
      .eq('user_id', userId)
      .order('created_at', { ascending: false });

    if (error) throw error;

    todos = ((data ?? []) as TodoRow[]).map(fromDatabase);
  } catch (error) {
    console.error('Failed to load tasks from Supabase:', error);
    storageError = true;
  } finally {
    storageReady = true;
  }
}


  onMount(() => {
    void loadTodos();
  });

async function logout() {
  const { error } = await supabase.auth.signOut();

  if (error) {
    console.error('Logout failed:', error);
    alert('Unable to log out. Please try again.');
    return;
  }

  window.location.href = '/login';
}

  function addTodo(event: SubmitEvent) {
    event.preventDefault();
    void createTodo();
  }

  async function createTodo() {
    const cleanTitle = title.trim();
    if (!cleanTitle) return;

    storageError = false;

    try {
      const userId = await getUserId();

      const { data, error } = await supabase
        .from('todos')
        .insert({
          user_id: userId,
          title: cleanTitle,
          description: description.trim(),
          completed: false
        })
        .select('id, title, description, completed, created_at, updated_at')
        .single();

      if (error) throw error;

      todos = [fromDatabase(data as TodoRow), ...todos];
      title = '';
      description = '';
    } catch (error) {
      console.error('Failed to add task:', error);
      storageError = true;
    }
  }

  function toggleTodo(id: Todo['id']) {
    void updateTodoStatus(id);
  }

  async function updateTodoStatus(id: Todo['id']) {
    const current = todos.find(todo => todo.id === id);
    if (!current) return;

    storageError = false;

    try {
      const userId = await getUserId();

      const { data, error } = await supabase
        .from('todos')
        .update({
          completed: !current.completed,
          updated_at: new Date().toISOString()
        })
        .eq('id', Number(id))
        .eq('user_id', userId)
        .select('id, title, description, completed, created_at, updated_at')
        .single();

      if (error) throw error;

      const updated = fromDatabase(data as TodoRow);
      todos = todos.map(todo => todo.id === id ? updated : todo);
    } catch (error) {
      console.error('Failed to update task status:', error);
      storageError = true;
    }
  }

  function startEdit(todo: Todo) {
    editingId = todo.id;
    editTitle = todo.title;
    editDescription = todo.description;
  }

  function saveEdit(id: Todo['id']) {
    void updateTodoDetails(id);
  }

  async function updateTodoDetails(id: Todo['id']) {
    const cleanTitle = editTitle.trim();
    if (!cleanTitle) return;

    storageError = false;

    try {
      const userId = await getUserId();

      const { data, error } = await supabase
        .from('todos')
        .update({
          title: cleanTitle,
          description: editDescription.trim(),
          updated_at: new Date().toISOString()
        })
        .eq('id', Number(id))
        .eq('user_id', userId)
        .select('id, title, description, completed, created_at, updated_at')
        .single();

      if (error) throw error;

      const updated = fromDatabase(data as TodoRow);
      todos = todos.map(todo => todo.id === id ? updated : todo);
      editingId = null;
    } catch (error) {
      console.error('Failed to edit task:', error);
      storageError = true;
    }
  }

  function cancelEdit() {
    editingId = null;
    editTitle = '';
    editDescription = '';
  }

  function deleteTodo(todo: Todo) {
    if (!confirm(`Delete "${todo.title}" permanently?`)) return;
    void removeTodo(todo);
  }

  async function removeTodo(todo: Todo) {
    storageError = false;

    try {
      const userId = await getUserId();

      const { error } = await supabase
        .from('todos')
        .delete()
        .eq('id', Number(todo.id))
        .eq('user_id', userId);

      if (error) throw error;

      todos = todos.filter(item => item.id !== todo.id);
      if (editingId === todo.id) cancelEdit();
    } catch (error) {
      console.error('Failed to delete task:', error);
      storageError = true;
    }
  }

  let completedCount = $derived(
    todos.filter(todo => todo.completed).length
  );

  let activeCount = $derived(
    todos.length - completedCount
  );

  let filteredTodos = $derived(
    todos.filter(todo => {
      if (filter === 'active') return !todo.completed;
      if (filter === 'completed') return todo.completed;
      return true;
    })
  );

  function formatDate(date: string) {
    return new Date(date).toLocaleDateString(undefined, {
      year: 'numeric',
      month: 'short',
      day: 'numeric'
    });
  }
</script>


<svelte:head>
  <title>My Todo List</title>
  <meta
    name="description"
    content="A simple task manager that saves your tasks in your browser."
  />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
</svelte:head>

<div class="app">
  <header class="topbar">

<button class="logout-button" onclick={logout}>
  Log out
</button>

    <a class="brand" href="#home" aria-label="My Todo List home">
      <span class="brand-icon">✓</span>
      <span>taskflow<span class="brand-dot">.</span></span>
    </a>
    <span class="local-badge">
      <span class="status-dot"></span>
      Synced with Supabase
    </span>
  </header>

  <main id="home">
    <section class="hero">
      <div class="eyebrow"><span></span> YOUR PERSONAL WORKSPACE</div>
      <h1>Make room for<br /><span>what matters.</span></h1>
      <p>Small steps every day lead to big results. What's on your list?</p>
    </section>

    {#if storageError}
      <div class="notice" role="alert">
        Browser storage is unavailable or full. Your latest changes may not persist after refreshing.
      </div>
    {/if}

    <section class="stats" aria-label="Task summary">
      <div class="stat-card">
        <span class="stat-label">TOTAL TASKS</span>
        <div class="stat-bottom">
          <strong>{todos.length}</strong>
          <span class="stat-icon purple">▤</span>
        </div>
      </div>
      <div class="stat-card">
        <span class="stat-label">IN PROGRESS</span>
        <div class="stat-bottom">
          <strong>{activeCount}</strong>
          <span class="stat-icon orange">◷</span>
        </div>
      </div>
      <div class="stat-card">
        <span class="stat-label">COMPLETED</span>
        <div class="stat-bottom">
          <strong>{completedCount}</strong>
          <span class="stat-icon green">✓</span>
        </div>
      </div>
    </section>

    <section class="composer">
      <div class="section-heading">
        <div>
          <h2>Add a new task</h2>
          <p>Write it down and get it done.</p>
        </div>
        <span class="plus">＋</span>
      </div>

      <form onsubmit={addTodo}>
        <label for="task-title">Task title <span>*</span></label>
        <input
          id="task-title"
          bind:value={title}
          placeholder="e.g. Finish my assignment"
          maxlength="200"
          required
        />

        <label for="task-description">Description <span class="optional">(optional)</span></label>
        <textarea
          id="task-description"
          bind:value={description}
          placeholder="Add a few details..."
          rows="2"
          maxlength="1000"
        ></textarea>

        <button class="add-button" type="submit">
          <span>＋</span> Add task
        </button>
      </form>
    </section>

    <section class="tasks-section">
      <div class="tasks-header">
        <div>
          <h2>Your tasks <span class="count">{filteredTodos.length}</span></h2>
          <p>Keep moving forward, one task at a time.</p>
        </div>
      </div>

      <div class="filters" aria-label="Filter tasks">
        <button
          class:chosen={filter === 'all'}
          aria-pressed={filter === 'all'}
          onclick={() => filter = 'all'}
        >All tasks <span>{todos.length}</span></button>
        <button
          class:chosen={filter === 'active'}
          aria-pressed={filter === 'active'}
          onclick={() => filter = 'active'}
        >Active <span>{activeCount}</span></button>
        <button
          class:chosen={filter === 'completed'}
          aria-pressed={filter === 'completed'}
          onclick={() => filter = 'completed'}
        >Completed <span>{completedCount}</span></button>
      </div>

      {#if !storageReady}
        <div class="empty-state">
          <div class="empty-icon">◌</div>
          <h3>Loading your tasks...</h3>
        </div>
      {:else if filteredTodos.length === 0}
        <div class="empty-state">
          <div class="empty-icon">
            {filter === 'completed' ? '✓' : filter === 'active' ? '✧' : '☷'}
          </div>
          <h3>
            {filter === 'completed'
              ? 'Nothing completed yet'
              : filter === 'active'
                ? 'You’re all caught up!'
                : 'Your list is a blank canvas'}
          </h3>
          <p>
            {filter === 'completed'
              ? 'Finish a task and it will appear here.'
              : filter === 'active'
                ? 'There are no active tasks right now.'
                : 'Add your first task above to get started.'}
          </p>
        </div>
      {:else}
        <div class="todo-list">
          {#each filteredTodos as todo (todo.id)}
            <article class:done={todo.completed} class="todo-card">
              {#if editingId === todo.id}
                <form class="edit-form" onsubmit={(event) => {
                  event.preventDefault();
                  saveEdit(todo.id);
                }}>
                  <label for={`edit-title-${todo.id}`}>Task title</label>
                  <input
                    id={`edit-title-${todo.id}`}
                    bind:value={editTitle}
                    maxlength="200"
                    required
                    autofocus
                  />
                  <label for={`edit-description-${todo.id}`}>Description</label>
                  <textarea
                    id={`edit-description-${todo.id}`}
                    bind:value={editDescription}
                    maxlength="1000"
                    rows="3"
                  ></textarea>
                  <div class="edit-actions">
                    <button type="button" class="secondary" onclick={cancelEdit}>Cancel</button>
                    <button type="submit" class="save-button">Save changes</button>
                  </div>
                </form>
              {:else}
                <div class="todo-check">
                  <input
                    type="checkbox"
                    id={`check-${todo.id}`}
                    checked={todo.completed}
                    onchange={() => toggleTodo(todo.id)}
                    aria-label={`Mark ${todo.title} as ${todo.completed ? 'active' : 'completed'}`}
                  />
                </div>
                <div class="todo-content">
                  <h3 class:completed-title={todo.completed}>{todo.title}</h3>
                  {#if todo.description}
                    <p class="todo-description">{todo.description}</p>
                  {/if}
                  <div class="todo-meta">
                    <span class="date-icon">◷</span>
                    Created {formatDate(todo.createdAt)}
                    {#if todo.updatedAt}
                      <span class="edited-tag">Edited</span>
                    {/if}
                  </div>
                </div>
                <div class="todo-actions">
                  <span class:completed-status={todo.completed} class="status">
                    {todo.completed ? 'Completed' : 'Active'}
                  </span>
                  <button
                    class="icon-button"
                    aria-label={`Edit ${todo.title}`}
                    title="Edit task"
                    onclick={() => startEdit(todo)}
                  >✎</button>
                  <button
                    class="icon-button delete-button"
                    aria-label={`Delete ${todo.title}`}
                    title="Delete task"
                    onclick={() => deleteTodo(todo)}
                  >⌫</button>
                </div>
              {/if}
            </article>
          {/each}
        </div>
      {/if}

      {#if todos.length > 0}
        <div class="progress-footer">
          <div class="progress-label">
            <span>Overall progress</span>
            <strong>{Math.round(completedCount / todos.length * 100)}%</strong>
          </div>
          <div
            class="progress-track"
            role="progressbar"
            aria-label="Task completion progress"
            aria-valuenow={completedCount}
            aria-valuemin="0"
            aria-valuemax={todos.length}
          >
            <div class="progress-fill" style={`width: ${completedCount / todos.length * 100}%`}></div>
          </div>
        </div>
      {/if}
    </section>

    <footer>
      <span>Made for a more focused day.</span>
      <span><span class="footer-dot"></span> {storageReady ? 'Ready to go' : 'Loading...'}</span>
    </footer>
  </main>
</div>

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    background: #f7f7fb;
    color: #25243a;
    font-family: Inter, ui-sans-serif, system-ui, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
    font-size: 14px;
    -webkit-font-smoothing: antialiased;
  }

  :global(button), :global(input), :global(textarea) {
    font: inherit;
  }

  :global(button:focus-visible), :global(input:focus-visible), :global(textarea:focus-visible), :global(a:focus-visible) {
    outline: 3px solid #b9adff;
    outline-offset: 3px;
  }

  .app { min-height: 100vh; }

  .topbar {
    height: 72px;
    padding: 0 max(24px, calc((100% - 920px) / 2));
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: #fff;
    border-bottom: 1px solid #ecebf3;
  }

  .brand {
    display: flex;
    gap: 10px;
    align-items: center;
    color: #282440;
    text-decoration: none;
    font-size: 21px;
    font-weight: 800;
    letter-spacing: -1px;
  }

  .brand-icon {
    display: grid;
    place-items: center;
    width: 34px;
    height: 34px;
    color: white;
    background: #6d58d9;
    border-radius: 11px;
    font-size: 20px;
  }

  .brand-dot { color: #8066f5; }

  .local-badge {
    display: flex;
    align-items: center;
    gap: 8px;
    color: #6c6a7c;
    font-size: 12px;
  }

  .status-dot, .footer-dot {
    width: 7px;
    height: 7px;
    display: inline-block;
    border-radius: 50%;
    background: #3dbb83;
  }

  main {
    width: min(100% - 40px, 760px);
    margin: 0 auto;
  }

  .hero { padding: 48px 0 28px; }

  .eyebrow {
    display: flex;
    align-items: center;
    gap: 9px;
    color: #7865d8;
    font-size: 10px;
    font-weight: 800;
    letter-spacing: 1.6px;
  }

  .eyebrow span {
    width: 20px;
    height: 2px;
    background: #8a75f4;
  }

  .hero h1 {
    margin: 17px 0 12px;
    font-size: clamp(35px, 7vw, 49px);
    line-height: 1.12;
    letter-spacing: -2.1px;
    font-weight: 800;
  }

  .hero h1 span { color: #7862df; }

  .hero > p, .section-heading p, .tasks-header p {
    color: #858398;
    margin: 0;
    line-height: 1.7;
  }

  .stats {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 14px;
    margin-bottom: 24px;
  }

  .stat-card {
    background: #fff;
    padding: 19px 20px 16px;
    border: 1px solid #eeedf5;
    border-radius: 14px;
    box-shadow: 0 3px 12px #29204a05;
  }

  .stat-label {
    color: #858398;
    font-size: 10px;
    font-weight: 750;
    letter-spacing: .9px;
  }

  .stat-bottom {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-top: 11px;
  }

  .stat-bottom strong { font-size: 29px; letter-spacing: -1px; }

  .stat-icon {
    display: grid;
    place-items: center;
    width: 35px;
    height: 35px;
    border-radius: 10px;
    font-size: 19px;
  }

  .purple { color: #8068e9; background: #f0edff; }
  .orange { color: #dc9b43; background: #fff4e3; }
  .green { color: #2eaa79; background: #e6f8f0; }

  .composer, .tasks-section {
    padding: 25px;
    background: #fff;
    border: 1px solid #eeedf5;
    border-radius: 16px;
    box-shadow: 0 5px 20px #29204a04;
  }

  .composer { margin-bottom: 24px; }

  .section-heading, .tasks-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 23px;
  }

  .section-heading h2, .tasks-header h2 {
    margin: 0 0 5px;
    font-size: 17px;
    letter-spacing: -.3px;
  }

  .section-heading p, .tasks-header p { font-size: 12px; }

  .plus {
    width: 38px;
    height: 38px;
    display: grid;
    place-items: center;
    border-radius: 11px;
    color: #7761de;
    background: #f2efff;
    font-size: 24px;
  }

  form label, .edit-form label {
    display: block;
    margin: 16px 0 8px;
    color: #48465d;
    font-size: 12px;
    font-weight: 700;
  }

  form label span { color: #e16b76; }
  form label .optional { color: #9b99aa; font-weight: 400; }

  input:not([type="checkbox"]), textarea {
    width: 100%;
    min-width: 0;
    border: 1px solid #e7e5f0;
    border-radius: 9px;
    padding: 12px 13px;
    background: #fdfdff;
    color: #29263f;
    transition: border-color .18s, box-shadow .18s;
  }

  input:not([type="checkbox"]):focus, textarea:focus {
    border-color: #8c79ec;
    box-shadow: 0 0 0 3px #8c79ec1c;
    outline: none;
  }

  input::placeholder, textarea::placeholder { color: #aaa8b8; }
  textarea { resize: vertical; line-height: 1.6; }

  .add-button, .save-button {
    border: 0;
    border-radius: 9px;
    background: #715bd8;
    color: white;
    font-weight: 700;
    cursor: pointer;
    transition: background .2s, transform .2s;
  }

  .add-button {
    margin-top: 17px;
    width: 100%;
    padding: 13px;
  }

  .add-button span { margin-right: 5px; font-size: 17px; }
  .add-button:hover, .save-button:hover { background: #5e48c5; }
  .add-button:active, .save-button:active { transform: scale(.99); }

  .tasks-section { padding-bottom: 20px; }
  .tasks-header { margin-bottom: 20px; }
  .tasks-header h2 { font-size: 19px; }

  .count {
    display: inline-grid;
    place-items: center;
    min-width: 23px;
    height: 23px;
    margin-left: 5px;
    border-radius: 7px;
    background: #f0edff;
    color: #725bd6;
    font-size: 11px;
    vertical-align: 2px;
  }

  .filters {
    display: flex;
    gap: 6px;
    padding: 5px;
    margin-bottom: 17px;
    border-radius: 10px;
    background: #f7f6fb;
  }

  .filters button {
    flex: 1;
    min-width: 0;
    padding: 10px 6px;
    border: 0;
    border-radius: 7px;
    background: transparent;
    color: #828095;
    font-size: 12px;
    font-weight: 600;
    cursor: pointer;
    transition: background .2s, color .2s;
  }

  .filters button span {
    margin-left: 4px;
    color: #aaa7bb;
    font-size: 11px;
  }

  .filters button.chosen {
    color: #6c55d4;
    background: white;
    box-shadow: 0 2px 6px #21154a0b;
  }

  .filters button.chosen span { color: #7967d9; }

  .todo-list { display: grid; gap: 10px; }

  .todo-card {
    display: flex;
    align-items: flex-start;
    gap: 13px;
    min-width: 0;
    padding: 17px 14px;
    border: 1px solid #eeedf5;
    border-radius: 12px;
    transition: border-color .2s, box-shadow .2s, background .2s;
  }

  .todo-card:hover {
    border-color: #d9d2ff;
    box-shadow: 0 4px 12px #33245a08;
  }

  .todo-card.done { background: #fcfcfe; }
  .todo-check { padding-top: 2px; }

  .todo-check input {
    appearance: none;
    width: 19px;
    height: 19px;
    display: grid;
    place-content: center;
    border: 1.5px solid #d5d2e3;
    border-radius: 6px;
    background: white;
    cursor: pointer;
  }

  .todo-check input:checked {
    border-color: #7863dc;
    background: #7863dc;
  }

  .todo-check input:checked::after {
    content: "✓";
    color: white;
    font-size: 13px;
    font-weight: 800;
  }

  .todo-content { flex: 1; min-width: 0; }

  .todo-content h3 {
    overflow-wrap: anywhere;
    margin: 0 0 6px;
    font-size: 14px;
    font-weight: 700;
    line-height: 1.5;
  }

  .completed-title {
    color: #9997a9;
    text-decoration: line-through;
  }

  .todo-description {
    overflow-wrap: anywhere;
    margin: 0 0 11px;
    color: #828095;
    font-size: 12px;
    line-height: 1.7;
    white-space: pre-wrap;
  }

  .todo-meta {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 5px;
    color: #a09daf;
    font-size: 10px;
  }

  .date-icon { font-size: 13px; }

  .edited-tag {
    margin-left: 5px;
    padding: 3px 6px;
    border-radius: 5px;
    background: #f2efff;
    color: #7964d9;
  }

  .todo-actions {
    display: flex;
    align-items: center;
    gap: 5px;
    flex-shrink: 0;
  }

  .status {
    padding: 5px 8px;
    margin-right: 3px;
    border-radius: 6px;
    color: #b07a2c;
    background: #fff5e6;
    font-size: 10px;
    font-weight: 700;
  }

  .completed-status {
    color: #259b6c;
    background: #e8f8f0;
  }

  .icon-button {
    display: grid;
    place-items: center;
    width: 30px;
    height: 30px;
    border: 1px solid transparent;
    border-radius: 7px;
    background: transparent;
    color: #9996ad;
    font-size: 17px;
    cursor: pointer;
    transition: background .2s, color .2s;
  }

  .icon-button:hover { background: #f0edff; color: #7058d4; }
  .delete-button:hover { background: #fff0f1; color: #d95360; }

  .empty-state {
    padding: 42px 15px;
    text-align: center;
    border: 1px dashed #e5e2f0;
    border-radius: 12px;
    background: #fdfcff;
  }

  .empty-icon {
    display: grid;
    place-items: center;
    width: 48px;
    height: 48px;
    margin: 0 auto 15px;
    border-radius: 15px;
    background: #f0edff;
    color: #7964d9;
    font-size: 25px;
  }

  .empty-state h3 { margin: 0 0 8px; font-size: 14px; }
  .empty-state p { margin: 0; color: #918ea3; font-size: 12px; line-height: 1.7; }

  .progress-footer {
    padding-top: 20px;
    margin-top: 22px;
    border-top: 1px solid #f0eef6;
  }

  .progress-label {
    display: flex;
    justify-content: space-between;
    margin-bottom: 10px;
    color: #858297;
    font-size: 11px;
  }

  .progress-label strong { color: #6e58d4; }

  .progress-track {
    height: 6px;
    overflow: hidden;
    border-radius: 10px;
    background: #eeecf5;
  }

  .progress-fill {
    height: 100%;
    border-radius: inherit;
    background: linear-gradient(90deg, #8e79ef, #6651d3);
    transition: width .3s ease;
  }

  .notice {
    margin-bottom: 20px;
    padding: 12px;
    border: 1px solid #f1d7a7;
    border-radius: 9px;
    background: #fff8e9;
    color: #805b1c;
    font-size: 12px;
    line-height: 1.6;
  }

  .edit-form { width: 100%; }
  .edit-form label:first-child { margin-top: 0; }

  .edit-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 15px;
  }

  .edit-actions button {
    padding: 9px 13px;
    border-radius: 8px;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
  }

  .secondary {
    border: 1px solid #e7e4f0;
    background: white;
    color: #6f6b81;
  }

  .secondary:hover { background: #f7f6fb; }

  .save-button { border: 0; }

  footer {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    padding: 24px 0 32px;
    color: #9996a9;
    font-size: 11px;
  }

  footer > span:last-child {
    display: flex;
    align-items: center;
    gap: 7px;
  }

  @media (max-width: 560px) {
    .topbar { height: 62px; padding: 0 18px; }
    .brand { font-size: 19px; }
    .local-badge { font-size: 10px; gap: 5px; }
    main { width: calc(100% - 28px); }
    .hero { padding: 36px 0 23px; }
    .hero h1 { letter-spacing: -1.5px; }
    .stats { gap: 8px; }
    .stat-card { padding: 13px 10px; border-radius: 11px; }
    .stat-label { font-size: 8px; letter-spacing: .3px; }
    .stat-bottom strong { font-size: 25px; }
    .stat-icon { width: 28px; height: 28px; font-size: 15px; }
    .composer, .tasks-section { padding: 17px 13px; border-radius: 13px; }
    .filters { gap: 2px; }
    .filters button { font-size: 11px; padding: 10px 3px; }
    .filters button span { margin-left: 2px; }
    .todo-card { gap: 9px; padding: 13px 9px; }
    .todo-actions { gap: 2px; flex-wrap: wrap; justify-content: flex-end; }
    .status { font-size: 9px; padding: 4px 6px; }
    .icon-button { width: 27px; height: 27px; }
  }

  @media (max-width: 370px) {
    .local-badge { max-width: 100px; }
    .todo-card { flex-wrap: wrap; }
    .todo-content { width: calc(100% - 35px); flex: none; }
    .todo-actions { width: 100%; justify-content: flex-end; }
    .stat-card { padding: 12px 8px; }
    .stat-label { font-size: 7px; }
  }

  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      scroll-behavior: auto !important;
      animation-duration: .01ms !important;
      transition-duration: .01ms !important;
    }

.logout-button {
  padding: 9px 13px;
  border: 1px solid #e7e5f0;
  border-radius: 9px;
  background: white;
  color: #6c55d4;
  font-weight: 700;
  cursor: pointer;
}

.logout-button:hover {
  background: #f2efff;
}
  }
</style>
