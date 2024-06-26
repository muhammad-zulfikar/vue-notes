<template>
  <Navbar />
  <SearchBar @update:modelValue="updateSearchQuery" />

  <Toast v-if="showToast" :message="toastMessage" />

  <div class="flex justify-between w-11/12 mx-auto mt-2 font-serif">
    <div>
      <button :onclick="startEditing"
        class="text-black dark:text-white hover:underline dark:hover:bg-transparent outline-none mr-4">Add New</button>
    </div>
    <div>
      <button @click="undoDelete"
        class="text-black dark:text-white hover:underline dark:hover:bg-transparent outline-none mr-4">Undo</button>
      <button @click="deleteAllNotes"
        class="text-black dark:text-white hover:underline dark:hover:bg-transparent outline-none">Delete All</button>
    </div>
  </div>

  <NoteForm v-if="editing" @closeForm="handleFormClose" />

  <NoteList :notes="localNotes" :searchQuery="searchQuery" :openNote="openNote" :removeNote="removeNote" />

  <NoteView v-if="selectedNote" :note="selectedNote" @close="closeNote" @save="saveNote" />
</template>

<script lang="ts" setup>
import { ref, watch, onMounted, nextTick } from 'vue';
import { useNotesStore, Note } from '@/stores/ProductStore';
import Navbar from './Navbar/Navbar.vue';
import SearchBar from './SearchBar/SearchBar.vue';
import NoteForm from './Notes/NoteForm.vue';
import NoteView from './Notes/NoteView.vue';
import Toast from './Toast/Toast.vue';
import NoteList from './Notes/NoteList.vue';

const showToast = ref(false);
const toastMessage = ref('');
const notesStore = useNotesStore();
const localNotes = ref<Note[]>([...notesStore.notes]);
const deletedNotes = ref<Note[]>([]);
const selectedNote = ref<Note | null>(null);
const searchQuery = ref('');
const undoStack = ref<Note[][]>([]);
const redoStack = ref<Note[][]>([]);
const editing = ref(false);
const defaultNotesDeleted = ref(false);

const updateSearchQuery = (query: string) => {
  searchQuery.value = query;
};

const title = ref('');
const content = ref('');

const startEditing = async (note: Note | null = null) => {
  editing.value = true;
  if (note) {
    title.value = note.title;
    content.value = note.content;
  } else {
    title.value = '';
    content.value = '';
  }
  await nextTick();
};

const handleFormClose = () => {
  editing.value = false;
};

watch(
  () => notesStore.notes,
  (newNotes) => {
    localNotes.value = [...newNotes];
  },
  { deep: true }
);

const saveNotesToLocalStorage = () => {
  localStorage.setItem('notes', JSON.stringify(localNotes.value));
};

onMounted(() => {
  const savedNotes = localStorage.getItem('notes');
  if (savedNotes) {
    localNotes.value = JSON.parse(savedNotes);
  } else {
    saveNotesToLocalStorage();
  }
});

watch(localNotes, saveNotesToLocalStorage, { deep: true });

const openNote = (index: number) => {
  selectedNote.value = localNotes.value[index];
};

const removeNote = (index: number) => {
  const deletedNote = localNotes.value.splice(index, 1)[0];
  deletedNotes.value.push(deletedNote);
  closeNote();
  showToast.value = true;
  toastMessage.value = 'Note deleted successfully!';
};

const closeNote = () => {
  selectedNote.value = null;
};

const saveNote = (note: Note) => {
  const currentTime = new Date().toLocaleString("sk-SK", {
    day: '2-digit',
    month: '2-digit',
    year: 'numeric',
    hour: 'numeric',
    minute: '2-digit',
    hour12: false,
  });
  note.timeCreated = currentTime;
  selectedNote.value = null;
  showToast.value = true;
  toastMessage.value = 'Note saved successfully!';
};

const deleteAllNotes = () => {
  undoStack.value.push([...localNotes.value]);
  redoStack.value = [];
  deletedNotes.value.push(...localNotes.value);
  localNotes.value = [];
  notesStore.setNotes([]);
  defaultNotesDeleted.value = true;
  showToast.value = true;
  toastMessage.value = 'All notes deleted successfully!';
};

const undoDelete = () => {
  if (deletedNotes.value.length > 0) {
    const lastDeletedNote = deletedNotes.value.pop();
    if (lastDeletedNote) {
      localNotes.value.push(lastDeletedNote);
      undoStack.value.push([...localNotes.value]);
      defaultNotesDeleted.value = false;
      showToast.value = true;
      toastMessage.value = 'Note restored successfully!';
    }
  }
};
</script>