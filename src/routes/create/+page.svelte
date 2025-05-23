<script lang="ts">
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { page } from '$app/stores';
  import { user } from '$lib/stores/user';
  import { searchUsers, npubToHex, isValidNpub, hexToNpub } from '$lib/services/vertex-search';
  import { publishFollowList, getFollowListById, deleteFollowList } from '$lib/services/follow-list.service';
  import { getProfileByPubkey } from '$lib/stores/user';
  import type { VertexSearchResult } from '$lib/services/vertex-search';
  import type { FollowList, FollowListEntry } from '$lib/types/follow-list';
  import PublicKeyDisplay from '$lib/components/PublicKeyDisplay.svelte';

  // Debug logging
  const DEBUG = true;
  const logDebug = (...args: any[]) => {
    if (DEBUG) console.log('[Create Page]', ...args);
  };

  // Form state
  let name = '';
  let coverImageUrl = '';
  let description = '';
  let searchQuery = '';
  let searching = false;
  let searchResults: VertexSearchResult[] = [];
  let selectedEntries: FollowListEntry[] = [];
  let submitting = false;
  let error = '';
  let editMode = false;
  let editId = '';
  let listId = '';
  let listEventId = '';
  let showDeleteConfirm = false;
  let deleting = false;
  let loading = false;
  
  // Validation state
  let nameValid = true;
  let entriesValid = true;
  
  onMount(async () => {
    // Check if we're in edit mode
    const editParam = $page.url.searchParams.get('edit');
    if (editParam) {
      editMode = true;
      editId = editParam;
      logDebug('Loading existing list for editing:', editId);
      loading = true;
      try {
        await loadExistingList(editParam);
      } finally {
        loading = false;
      }
    }
  });
  
  // Handle delete button click
  function handleDelete() {
    showDeleteConfirm = true;
  }
  
  // Handle delete confirmation
  async function confirmDelete() {
    if (!listEventId) return;
    
    deleting = true;
    try {
      const deleted = await deleteFollowList(listEventId);
      if (deleted) {
        // Navigate back to home
        goto('/');
      } else {
        // Handle error
        error = 'Failed to delete follow list';
        showDeleteConfirm = false;
      }
    } catch (err: any) {
      console.error('Error deleting follow list:', err);
      error = `Error deleting: ${err.message || 'An unknown error occurred'}`;
      showDeleteConfirm = false;
    } finally {
      deleting = false;
    }
  }
  
  // Cancel deletion
  function cancelDelete() {
    showDeleteConfirm = false;
  }
  
  // Load an existing list for editing
  async function loadExistingList(id: string) {
    try {
      const list = await getFollowListById(id);
      if (!list) {
        error = 'Could not find the follow list to edit';
        editMode = false;
        return;
      }
      
      // Check if the user is the author
      if (!$user || $user.pubkey !== list.pubkey) {
        error = 'You are not authorized to edit this follow list';
        editMode = false;
        return;
      }
      
      // Load the list data into the form
      name = list.name;
      coverImageUrl = list.coverImageUrl;
      description = list.description || '';
      listId = list.id;
      listEventId = list.eventId;
      selectedEntries = [...list.entries];

      // reactively get profile info for each entry
      selectedEntries.forEach(async (entry) => {
        const profile = await getProfileByPubkey(entry.pubkey);
        entry.name = profile.name;
        entry.picture = profile.picture;
        entry.bio = profile.bio;
        entry.nip05 = profile.nip05;
      });
      
      logDebug('Loaded existing list for editing:', { id, name, entries: selectedEntries.length });
    } catch (err: any) {
      console.error('Error loading follow list for editing:', err);
      error = `Error loading list: ${err.message || 'Unknown error'}`;
      editMode = false;
    }
  }
  
  // Handle user search
  async function handleSearch() {
    if (!searchQuery.trim()) return;
    
    searching = true;
    searchResults = [];
    error = '';
    
    try {
      // Check if input is a valid npub
      if (isValidNpub(searchQuery)) {
        // Convert npub to hex
        const pubkey = await npubToHex(searchQuery);

        if (!pubkey) {
          error = 'Invalid npub';
          return;
        }
        
        // Get profile
        const profile = await getProfileByPubkey(pubkey);
        
        // Create a single search result
        searchResults = [{
          pubkey,
          rank: 0,
          name: profile.name || 'Unknown',
          picture: profile.picture || '',
        }];
      } else {
        // Assume it's a NIP-05 domain
        let domain = searchQuery.trim().toLowerCase();
        domain = domain.replace(/^https?:\/\//, ''); // Remove http(s)://
        domain = domain.split('/')[0]; // Remove any paths
        
        // Make request to .well-known/nostr.json with CORS handling
        const response = await fetch(`https://${domain}/.well-known/nostr.json`, {
          mode: 'cors',
          headers: {
            'Accept': 'application/json',
            'Origin': window.location.origin
          }
        }).catch((err) => {
          // Check if the error is a CORS error
          if (err.message.includes('Failed to fetch') || err.message.includes('CORS')) {
            throw new Error(`CORS Error: The domain ${domain} is blocking requests from this website. Send the domain owner https://github.com/nostr-protocol/nips/blob/master/05.md#allowing-access-from-javascript-apps.`);
          }
          throw err;
        });
        
        if (!response.ok) {
          if (response.status === 404) {
            throw new Error('NIP-05 file not found. Make sure the domain supports NIP-05.');
          } else if (response.status === 403) {
            throw new Error('Access denied. The server is blocking requests to the NIP-05 file.');
          } else {
            throw new Error(`HTTP error! status: ${response.status}`);
          }
        }
        
        const data = await response.json();
        
        if (!data.names) {
          throw new Error('Invalid nostr.json format: missing names field');
        }
        
        // Get all pubkeys from the names object and ensure they are strings
        const pubkeys = Object.values(data.names).filter((key): key is string => typeof key === 'string');
        
        if (pubkeys.length === 0) {
          throw new Error('No valid pubkeys found in the NIP-05 list');
        }
        
        // Add each pubkey to the list
        for (const pubkey of pubkeys) {
          try {
            const profile = await getProfileByPubkey(pubkey);
            addEntry({
              pubkey,
              rank: 0,
              name: profile.name || 'Unknown',
              picture: profile.picture || '',
            });
          } catch (err) {
            console.error(`Error processing pubkey ${pubkey}:`, err);
          }
        }
        
        // Clear the search field
        searchQuery = '';
        return;
      }
      
      logDebug(`Search returned ${searchResults.length} results`);
    } catch (err: any) {
      console.error('Error searching:', err);
      error = `Search error: ${err.message || 'An unknown error occurred'}`;
    } finally {
      searching = false;
    }
  }
  
  // Add a search result to the selected entries
  function addEntry(result: VertexSearchResult) {
    // Check if already added
    if (selectedEntries.some(entry => entry.pubkey === result.pubkey)) {
      logDebug('Entry already in list:', result.pubkey);
      return;
    }
    
    logDebug('Adding entry to list:', result);
    
    // Add to selections
    selectedEntries = [
      ...selectedEntries, 
      {
        pubkey: result.pubkey,
        name: result.name,
        picture: result.picture
      }
    ];
    
    // Clear search
    searchQuery = '';
    searchResults = [];
  }
  
  // Remove a selected entry
  function removeEntry(index: number) {
    selectedEntries = selectedEntries.filter((_, i) => i !== index);
  }
  
  // Move entry up in the list
  function moveEntryUp(index: number) {
    if (index <= 0) return; // Can't move up if already at the top
    
    const newEntries = [...selectedEntries];
    const temp = newEntries[index];
    newEntries[index] = newEntries[index - 1];
    newEntries[index - 1] = temp;
    
    selectedEntries = newEntries;
  }
  
  // Move entry down in the list
  function moveEntryDown(index: number) {
    if (index >= selectedEntries.length - 1) return; // Can't move down if already at the bottom
    
    const newEntries = [...selectedEntries];
    const temp = newEntries[index];
    newEntries[index] = newEntries[index + 1];
    newEntries[index + 1] = temp;
    
    selectedEntries = newEntries;
  }
  
  // Handle form submission
  async function handleSubmit() {
    // Reset validation
    nameValid = !!name.trim();
    entriesValid = selectedEntries.length > 0;
    
    logDebug('Form submission - validation:', { nameValid, entriesValid });
    
    // Check validation
    if (!nameValid || !entriesValid) {
      error = 'Please fix the validation errors and try again.';
      logDebug('Validation failed:', error);
      return;
    }
    
    submitting = true;
    error = '';
    
    logDebug('Publishing follow list:', { name, description, entries: selectedEntries.length, editMode });
    
    try {
      // Publish the follow list (same method for create and edit)
      const id = await publishFollowList(name, coverImageUrl, selectedEntries, editMode ? listId : undefined, description);
      logDebug('Published with ID:', id);
      
      if (id) {
        // Navigate to the new follow list
        logDebug('Redirecting to follow list page');
        goto(`/d/${id}`);
      } else {
        error = 'Failed to publish follow list. Please try again.';
        logDebug('Publish failed - no ID returned');
      }
    } catch (err: any) {
      console.error('Error publishing follow list:', err);
      logDebug('Publish error:', err);
      error = `Error publishing: ${err.message || 'An unknown error occurred'}`;
    } finally {
      submitting = false;
    }
  }
</script>

<div class="container py-10">
  <div class="max-w-2xl mx-auto">
    <div class="flex justify-between items-center mb-8">
      <h1 class="text-3xl font-bold text-gray-900">{editMode ? 'Edit' : 'Create'} Follow Pack</h1>
      <a href="/" class="btn btn-secondary">Cancel</a>
    </div>
    
    {#if !$user}
      <div class="bg-yellow-50 border border-yellow-200 rounded-md p-6 text-center">
        <h2 class="text-xl font-medium text-yellow-800 mb-2">Login Required</h2>
        <p class="text-yellow-700 mb-4">
          You need to be logged in with a NIP-07 extension to create a follow list.
        </p>
        <a href="/" class="btn btn-primary">Back to Home</a>
      </div>
    {:else if editMode && loading}
      <div class="flex justify-center py-12">
        <div class="animate-spin rounded-full h-12 w-12 border-t-2 border-b-2 border-purple-500"></div>
      </div>
    {:else}
      {#if error}
        <div class="bg-red-50 border border-red-200 rounded-md p-4 mb-6 text-red-700">
          {error}
        </div>
      {/if}
      
      <!-- Delete Confirmation Modal -->
      {#if showDeleteConfirm}
        <div class="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
          <div class="bg-white rounded-lg p-6 max-w-md w-full">
            <h3 class="text-xl font-bold text-gray-900 mb-4">Delete Follow Pack</h3>
            <p class="text-gray-700 mb-6">
              Are you sure you want to delete the follow list "{name}"? This action cannot be undone.
            </p>
            <div class="flex justify-end space-x-3">
              <button 
                on:click={cancelDelete} 
                class="btn btn-secondary"
                disabled={deleting}
              >
                Cancel
              </button>
              <button 
                on:click={confirmDelete} 
                class="btn btn-error"
                disabled={deleting}
              >
                {deleting ? 'Deleting...' : 'Delete Forever'}
              </button>
            </div>
          </div>
        </div>
      {/if}
      
      <form 
        on:submit|preventDefault={handleSubmit} 
        on:keydown={(e) => {
          if (e.key === 'Enter' && e.target instanceof HTMLElement && e.target.tagName !== 'TEXTAREA') {
            e.preventDefault();
          }
        }}
        class="bg-white shadow-sm rounded-lg overflow-hidden"
      >
        <div class="p-6 border-b">
          <h2 class="text-xl font-medium mb-6">Follow Pack Details</h2>
          
          <!-- Name field -->
          <div class="mb-4">
            <label for="name" class="block text-sm font-medium text-gray-700 mb-1">
              List Name <span class="text-red-500">*</span>
            </label>
            <input
              type="text"
              id="name"
              bind:value={name}
              on:blur={() => nameValid = !!name.trim()}
              class="w-full px-3 py-2 border {!nameValid ? 'border-red-500' : 'border-gray-300'} rounded-md shadow-sm focus:ring-purple-500 focus:border-purple-500"
              placeholder="E.g., Nostr Developers to Follow"
            />
            {#if !nameValid}
              <p class="mt-1 text-sm text-red-600">Name is required</p>
            {/if}
          </div>
          
          <!-- Cover image URL field -->
          <div class="mb-6">
            <label for="coverImageUrl" class="block text-sm font-medium text-gray-700 mb-1">
              Cover Image URL
            </label>
            <input
              type="url"
              id="coverImageUrl"
              bind:value={coverImageUrl}
              class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-purple-500 focus:border-purple-500"
              placeholder="https://example.com/image.jpg"
            />
            <p class="mt-1 text-xs text-gray-500">Optional: URL to an image that represents this list</p>
          </div>
          
          <!-- Description field -->
          <div class="mb-6">
            <label for="description" class="block text-sm font-medium text-gray-700 mb-1">
              Description
            </label>
            <textarea
              id="description"
              bind:value={description}
              class="w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-purple-500 focus:border-purple-500"
              placeholder="Enter a description for this follow list"
            ></textarea>
          </div>
          
          {#if coverImageUrl}
            <div class="mb-6">
              <p class="block text-sm font-medium text-gray-700 mb-2">Cover Image Preview</p>
              <div class="h-36 bg-gray-100 rounded-md overflow-hidden">
                <img 
                  src={coverImageUrl} 
                  alt="Cover Preview" 
                  class="w-full h-full object-cover" 
                  on:error={(e) => {
                    (e.target as HTMLImageElement).src = 'https://www.gravatar.com/avatar/00000000000000000000000000000000?d=mp&f=y';
                  }} 
                />
              </div>
            </div>
          {/if}
        </div>
        
        <div class="p-6 border-b">
          <h2 class="text-xl font-medium mb-6">Add Users to Follow Pack</h2>
          
          <!-- Search field -->
          <div class="mb-4">
            <label for="search" class="block text-sm font-medium text-gray-700 mb-1">
              Add Users to Follow Pack
            </label>
            <div class="flex">
              <input
                type="text"
                id="search"
                bind:value={searchQuery}
                class="flex-1 px-3 py-2 border border-gray-300 rounded-l-md shadow-sm focus:ring-purple-500 focus:border-purple-500"
                placeholder="Enter npub or NIP-05 domain (e.g. npub1... or bitcoinveterans.org)"
                on:keydown={e => {
                  if (e.key === 'Enter') {
                    e.preventDefault();
                    handleSearch();
                  }
                }}
              />
              <button
                type="button"
                on:click={handleSearch}
                disabled={searching || !searchQuery.trim()}
                class="px-4 py-2 bg-purple-600 text-white rounded-r-md hover:bg-purple-700 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-purple-500 disabled:opacity-50"
              >
                {searching ? 'Searching...' : 'Add'}
              </button>
            </div>
            <p class="mt-1 text-xs text-gray-500">
              Enter a nostr npub or NIP-05 domain to add users to your list
            </p>
            
            {#if error}
              <p class="mt-2 text-sm text-red-600">{error}</p>
            {/if}
          </div>
          
          <!-- Search results -->
          {#if searchResults.length > 0}
            <div class="mb-6 border border-gray-200 rounded-md overflow-hidden">
              <ul class="divide-y divide-gray-200">
                {#each searchResults as result}
                  <li class="p-3 flex items-center justify-between hover:bg-gray-50">
                    <div class="flex items-center">
                      <img 
                        src={result.picture || 'https://www.gravatar.com/avatar/00000000000000000000000000000000?d=mp&f=y'} 
                        alt={result.name || 'User'} 
                        class="w-8 h-8 rounded-full object-cover mr-3"
                      />
                      <div>
                        <p class="font-medium">{result.name || 'Unknown User'}</p>
                        <PublicKeyDisplay pubkey={result.pubkey} />
                      </div>
                    </div>
                    <button
                      type="button"
                      on:click={() => addEntry(result)}
                      class="text-sm text-purple-600 hover:text-purple-800"
                    >
                      Add to List
                    </button>
                  </li>
                {/each}
              </ul>
            </div>
          {/if}
          
          <!-- Selected entries -->
          <div>
            <div class="flex justify-between items-center mb-2">
              <h3 class="text-lg font-medium">Selected Users ({selectedEntries.length})</h3>
              {#if !entriesValid}
                <p class="text-sm text-red-600">Add at least one user to your list</p>
              {/if}
            </div>
            
            {#if selectedEntries.length === 0}
              <div class="bg-gray-50 border border-gray-200 rounded-md p-4 text-center text-gray-500">
                No users selected yet. Search for users to add them to your list.
              </div>
            {:else}
              <ul class="border border-gray-200 rounded-md divide-y divide-gray-200">
                {#each selectedEntries as entry, i}
                  <li class="p-3 flex items-center justify-between">
                    <div class="flex items-center">
                      <img 
                        src={entry.picture || 'https://www.gravatar.com/avatar/00000000000000000000000000000000?d=mp&f=y'} 
                        alt={entry.name || 'User'} 
                        class="w-8 h-8 rounded-full object-cover mr-3"
                      />
                      <div>
                        <p class="font-medium">{entry.name || 'Unknown User'}</p>
                        {#if entry.nip05}
                          <p class="text-xs text-gray-500">
                            {entry.nip05}
                          </p>
                        {/if}
                        <PublicKeyDisplay pubkey={entry.pubkey} />
                      </div>
                    </div>
                    <div class="flex items-center space-x-2">
                      <div class="flex flex-col mr-3">
                        <button
                          type="button"
                          on:click={() => moveEntryUp(i)}
                          disabled={i === 0}
                          class="text-gray-500 hover:text-purple-600 disabled:opacity-30 disabled:hover:text-gray-500 focus:outline-none"
                          title="Move up"
                        >
                          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                            <path fill-rule="evenodd" d="M14.707 12.707a1 1 0 01-1.414 0L10 9.414l-3.293 3.293a1 1 0 01-1.414-1.414l4-4a1 1 0 011.414 0l4 4a1 1 0 010 1.414z" clip-rule="evenodd" />
                          </svg>
                        </button>
                        <button
                          type="button"
                          on:click={() => moveEntryDown(i)}
                          disabled={i === selectedEntries.length - 1}
                          class="text-gray-500 hover:text-purple-600 disabled:opacity-30 disabled:hover:text-gray-500 focus:outline-none"
                          title="Move down"
                        >
                          <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" viewBox="0 0 20 20" fill="currentColor">
                            <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 011.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
                          </svg>
                        </button>
                      </div>
                      <button
                        type="button"
                        on:click={() => removeEntry(i)}
                        class="text-sm text-red-600 hover:text-red-800"
                      >
                        Remove
                      </button>
                    </div>
                  </li>
                {/each}
              </ul>
            {/if}
          </div>
        </div>
        
        <div class="p-6 flex justify-between">
          {#if editMode}
            <button
              type="button"
              on:click={handleDelete}
              class="btn btn-error px-6 py-3 text-base"
            >
              Delete List
            </button>
          {:else}
            <div></div>
          {/if}
          
          <button
            type="submit"
            disabled={submitting}
            class="btn btn-primary px-6 py-3 text-base {submitting ? 'opacity-70' : ''}"
          >
            {submitting ? (editMode ? 'Updating...' : 'Publishing...') : (editMode ? 'Update' : 'Publish') + ' Follow Pack'}
          </button>
        </div>
      </form>
    {/if}
  </div>
</div> 