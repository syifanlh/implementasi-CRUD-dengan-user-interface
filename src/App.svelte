<script>
  import { onMount } from 'svelte';
  import { createClient } from '@supabase/supabase-js';

  const supabaseUrl = (import.meta.env.VITE_SUPABASE_URL || '').trim();
  const supabaseAnonKey = (import.meta.env.VITE_SUPABASE_ANON_KEY || '').trim();
  const hasValidSupabaseConfig = /^https:\/\/[a-z0-9-]+\.supabase\.co(\.cn)?\/?$/i.test(supabaseUrl) && Boolean(supabaseAnonKey);
  const supabase = hasValidSupabaseConfig ? createClient(supabaseUrl, supabaseAnonKey) : null;

  let count = 0;
  let name = 'ali';
  let phone = '87463747';
  let newName = '';
  let newPhone = '';
  let editingId = null;
  let isLoading = false;
  let statusMessage = supabase ? 'Connected to Supabase' : 'Using local mode. Add your Supabase URL and key in .env.';

  let contacts = [
    { id: 1, name: 'ali', phone: '87463747' },
    { id: 2, name: 'khalda', phone: '123624687' },
    { id: 3, name: 'aliya', phone: '53584699' }
  ];

  function updateSummary(target = contacts[contacts.length - 1] ?? contacts[0]) {
    const current = target ?? null;
    if (current) {
      name = current.name;
      phone = current.phone;
    } else {
      name = 'No data';
      phone = 'Add contact';
    }
  }

  async function loadContacts() {
    if (!supabase) return;

    isLoading = true;
    statusMessage = 'Loading contacts...';

    const { data, error } = await supabase.from('contacts').select('*').order('id', { ascending: true });

    isLoading = false;

    if (error) {
      statusMessage = 'Supabase error: ' + error.message;
      console.error('loadContacts error:', error.message);
      return;
    }

    if (data && data.length > 0) {
      contacts = data.map((item) => ({ id: item.id, name: item.name, phone: item.phone }));
      updateSummary(contacts[contacts.length - 1]);
    } else {
      contacts = [];
      updateSummary();
    }

    statusMessage = 'Connected to Supabase';
  }

  async function handleSubmit() {
    const trimmedName = newName.trim();
    const trimmedPhone = newPhone.trim();

    if (!trimmedName || !trimmedPhone) return;

    if (supabase) {
      isLoading = true;
      statusMessage = editingId !== null ? 'Saving changes...' : 'Adding contact...';

      const payload = { name: trimmedName, phone: trimmedPhone };

      if (editingId !== null) {
        const { error } = await supabase.from('contacts').update(payload).eq('id', editingId);
        if (error) {
          statusMessage = 'Update failed: ' + error.message;
          isLoading = false;
          return;
        }

        const updated = contacts.find((contact) => contact.id === editingId);
        if (updated) {
          updateSummary({ ...updated, name: trimmedName, phone: trimmedPhone });
        }
      } else {
        const { data, error } = await supabase.from('contacts').insert(payload).select().single();
        if (error) {
          statusMessage = 'Insert failed: ' + error.message;
          isLoading = false;
          return;
        }

        if (data) {
          updateSummary(data);
        }
      }

      await loadContacts();
    } else {
      if (editingId !== null) {
        const updatedContact = contacts.find((contact) => contact.id === editingId);
        contacts = contacts.map((contact) =>
          contact.id === editingId ? { ...contact, name: trimmedName, phone: trimmedPhone } : contact
        );
        updateSummary(updatedContact ? { ...updatedContact, name: trimmedName, phone: trimmedPhone } : null);
      } else {
        const nextId = contacts.length ? Math.max(...contacts.map((contact) => contact.id)) + 1 : 1;
        const created = { id: nextId, name: trimmedName, phone: trimmedPhone };
        contacts = [...contacts, created];
        updateSummary(created);
      }
      statusMessage = 'Saved locally';
    }

    editingId = null;
    newName = '';
    newPhone = '';
    isLoading = false;
  }

  function handleClickMe() {
    count += 1;
  }

  function startEdit(contact) {
    editingId = contact.id;
    newName = contact.name;
    newPhone = contact.phone;
  }

  async function deleteContact(id) {
    if (supabase) {
      isLoading = true;
      statusMessage = 'Deleting contact...';
      const { error } = await supabase.from('contacts').delete().eq('id', id);
      isLoading = false;

      if (error) {
        statusMessage = 'Delete failed: ' + error.message;
        return;
      }

      await loadContacts();
    } else {
      contacts = contacts.filter((contact) => contact.id !== id);
      updateSummary();
      statusMessage = 'Deleted locally';
    }

    if (editingId === id) {
      editingId = null;
      newName = '';
      newPhone = '';
    }
  }

  onMount(() => {
    if (supabase) {
      loadContacts();
    } else {
      updateSummary();
    }
  });
</script>

<div class="app-shell">
  <div class="panel">
    <h1 class="title">Name: <span>{name}</span></h1>
    <h1 class="title">Phone: <span>{phone}</span></h1>

    <div class="count">Count: {count}</div>
    <div class="status">{statusMessage}</div>

    <button class="click-btn" type="button" on:click={handleClickMe}>Click Me</button>

    <form class="form-row" on:submit|preventDefault={handleSubmit}>
      <input bind:value={newName} type="text" placeholder="Enter your name" />
      <input bind:value={newPhone} type="text" placeholder="Enter your phone number" />
      <button type="submit" class="submit-btn" disabled={isLoading}>
        {editingId !== null ? 'Save' : 'Submit'}
      </button>
    </form>

    <table class="contact-table">
      <thead>
        <tr>
          <th>No</th>
          <th>Nama</th>
          <th>Nomor Telepon</th>
          <th>Aksi</th>
        </tr>
      </thead>
      <tbody>
        {#each contacts as contact, index (contact.id)}
          <tr>
            <td>{index + 1}</td>
            <td>{contact.name}</td>
            <td>{contact.phone}</td>
            <td class="actions">
              <button class="edit-btn" type="button" on:click={() => startEdit(contact)}>Edit</button>
              <button class="delete-btn" type="button" on:click={() => deleteContact(contact.id)}>Hapus</button>
            </td>
          </tr>
        {/each}
      </tbody>
    </table>
  </div>
</div>

<style>
  :global(html, body) {
    margin: 0;
    min-height: 100%;
    background: #050d14;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  }

  :global(body) {
    min-height: 100vh;
  }

  :global(*) {
    box-sizing: border-box;
  }

  :global(button),
  :global(input) {
    font: inherit;
  }

  :global(#app) {
    min-height: 100vh;
  }

  .app-shell {
    min-height: 100vh;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    padding-top: 30px;
  }

  .panel {
    width: min(860px, calc(100vw - 64px));
    text-align: center;
  }

  .title {
    margin: 0;
    font-size: clamp(2.5rem, 4vw, 5.1rem);
    font-weight: 900;
    line-height: 0.9;
    letter-spacing: -0.08em;
    color: #ff5d14;
    text-shadow: 0 0 12px rgba(255, 110, 58, 0.18);
  }

  .count {
    margin-top: 14px;
    color: #53c0ff;
    font-size: clamp(1.2rem, 2vw, 2rem);
    font-weight: 700;
    letter-spacing: -0.04em;
  }

  .status {
    margin-top: 8px;
    color: rgba(255, 255, 255, 0.7);
    font-size: 0.85rem;
    min-height: 1.2em;
  }

  .click-btn {
    width: 100%;
    margin-top: 18px;
    margin-bottom: 16px;
    padding: 12px 18px;
    border: none;
    border-radius: 6px;
    background: rgba(255, 255, 255, 0.08);
    color: #f5f5f5;
    font-size: 1.05rem;
    font-weight: 700;
    cursor: pointer;
  }

  .form-row {
    display: grid;
    grid-template-columns: 1fr 1fr auto;
    gap: 16px;
    align-items: center;
    margin-bottom: 24px;
  }

  .form-row input {
    width: 100%;
    box-sizing: border-box;
    padding: 14px 16px;
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 6px;
    background: rgba(255, 255, 255, 0.02);
    color: #f1f5f9;
    font-size: 1.02rem;
  }

  .form-row input::placeholder {
    color: rgba(255, 255, 255, 0.42);
  }

  .submit-btn {
    padding: 14px 24px;
    border: none;
    border-radius: 6px;
    background: #f3f4f6;
    color: #1b1d21;
    font-size: 1rem;
    font-weight: 700;
    cursor: pointer;
  }

  .submit-btn:disabled {
    opacity: 0.7;
    cursor: wait;
  }

  .contact-table {
    width: 100%;
    border-collapse: collapse;
    background: rgba(27, 30, 35, 0.82);
    box-shadow: inset 0 0 0 1px rgba(255, 255, 255, 0.08);
  }

  .contact-table th,
  .contact-table td {
    padding: 18px 16px;
    border: 1px solid rgba(255, 255, 255, 0.08);
    text-align: left;
    color: #f0f2f5;
  }

  .contact-table th {
    font-size: 1.06rem;
    font-weight: 800;
    color: #ff6c2c;
    background: rgba(255, 255, 255, 0.02);
  }

  .contact-table td {
    font-size: 1.12rem;
    font-weight: 600;
    color: rgba(255, 255, 255, 0.9);
  }

  .actions {
    display: flex;
    gap: 10px;
    align-items: center;
    justify-content: flex-start;
    min-width: 170px;
  }

  .edit-btn,
  .delete-btn {
    border: none;
    border-radius: 5px;
    padding: 8px 12px;
    font-size: 0.95rem;
    font-weight: 700;
    cursor: pointer;
  }

  .edit-btn {
    background: rgba(255, 255, 255, 0.12);
    color: #f5f5f5;
  }

  .delete-btn {
    background: #ef4444;
    color: #fff;
  }

  @media (max-width: 700px) {
    .panel {
      width: min(88vw, 600px);
    }

    .form-row {
      grid-template-columns: 1fr;
    }

    .contact-table {
      display: block;
      overflow-x: auto;
    }
  }
</style>
