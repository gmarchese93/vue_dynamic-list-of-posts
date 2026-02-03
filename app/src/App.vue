<template>
  <div class="section">
    <h1 class="title">Posts</h1>

    <button class="button is-primary mb-4" @click="openCreatePost">
      Create new post
    </button>

    <!-- POSTS LOADING -->
    <div v-if="postsLoading">Loading...</div>

    <!-- POSTS ERROR -->
    <div v-else-if="postsError" class="notification is-danger">
      Failed to load posts
    </div>

    <!-- NO POSTS -->
    <div v-else-if="posts.length === 0" class="notification is-warning">
      No posts yet
    </div>

    <!-- POSTS TABLE -->
    <table v-else class="table is-fullwidth">
      <thead>
        <tr>
          <th>ID</th>
          <th>Title</th>
        </tr>
      </thead>
      <tbody>
        <tr
          v-for="post in posts"
          :key="post.id"
          @click="selectPost(post)"
          style="cursor:pointer"
        >
          <td>{{ post.id }}</td>
          <td>{{ post.title }}</td>
        </tr>
      </tbody>
    </table>
  </div>

  <!-- SIDEBAR -->
  <div class="sidebar" :class="{ 'Sidebar--open': sidebarOpen }">
    <div class="sidebar-content">

      <!-- CREATE / EDIT POST -->
      <div v-if="isCreating || isEditing">
        <h2 class="title is-4">
          {{ isEditing ? 'Edit post' : 'Create post' }}
        </h2>

        <div class="field">
          <label class="label">Title</label>
          <input
            class="input"
            v-model="postForm.title"
            :class="{ 'is-danger': postFormSubmitted && !postForm.title }"
          />
        </div>

        <div class="field">
          <label class="label">Body</label>
          <textarea
            class="textarea"
            v-model="postForm.body"
            :class="{ 'is-danger': postFormSubmitted && !postForm.body }"
          />
        </div>

        <div class="buttons">
          <button class="button is-primary" @click="savePost">
            {{ isEditing ? 'Save' : 'Create' }}
          </button>
          <button class="button" @click="closeSidebar">Cancel</button>
        </div>
      </div>

      <!-- POST PREVIEW -->
      <div v-else-if="selectedPost">
        <h2 class="title is-4">Post preview</h2>

        <p><strong>{{ selectedPost.title }}</strong></p>
        <p class="mt-2">{{ selectedPost.body }}</p>

        <div class="buttons mt-3">
          <button class="button is-small" @click="startEdit">Edit</button>
          <button class="button is-small is-danger" @click="deletePost">
            Delete
          </button>
        </div>

        <hr />

        <!-- COMMENTS -->
        <h3 class="title is-5">Comments</h3>

        <div v-if="commentsLoading">Loading comments...</div>

        <div v-else-if="commentsError" class="notification is-danger">
          Failed to load comments
        </div>

        <div v-else-if="comments.length === 0" class="notification is-warning">
          No comments yet
        </div>

        <div v-else>
          <div v-for="c in comments" :key="c.id" class="box">
            <p class="has-text-weight-bold">
              {{ c.name }}
              <span class="is-size-7 has-text-grey">({{ c.email }})</span>
            </p>
            <p>{{ c.body }}</p>
            <button
              class="button is-small is-danger mt-2"
              @click="deleteComment(c)"
            >
              Delete
            </button>
          </div>
        </div>

        <!-- WRITE COMMENT -->
        <button
          v-if="!showCommentForm"
          class="button is-link mt-3"
          @click="showCommentForm = true"
        >
          Write a comment
        </button>

        <div v-else class="mt-3">
          <div class="field">
            <input
              class="input"
              placeholder="Name"
              v-model="commentForm.name"
              :class="{ 'is-danger': commentSubmitted && !commentForm.name }"
            />
          </div>

          <div class="field">
            <input
              class="input"
              placeholder="Email"
              v-model="commentForm.email"
              :class="{ 'is-danger': commentSubmitted && !commentForm.email }"
            />
          </div>

          <div class="field">
            <textarea
              class="textarea"
              placeholder="Comment"
              v-model="commentForm.body"
              :class="{ 'is-danger': commentSubmitted && !commentForm.body }"
            />
          </div>

          <div class="buttons">
            <button
              class="button is-primary"
              :class="{ 'is-loading': commentSubmitting }"
              @click="submitComment"
            >
              Submit
            </button>
            <button class="button" @click="clearCommentForm">Clear</button>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed } from 'vue';

const USER_ID = 1;

const posts = ref([]);
const postsLoading = ref(false);
const postsError = ref(false);

const selectedPost = ref(null);
const comments = ref([]);
const commentsLoading = ref(false);
const commentsError = ref(false);

const isCreating = ref(false);
const isEditing = ref(false);

const postForm = ref({ title: '', body: '' });
const postFormSubmitted = ref(false);

const showCommentForm = ref(false);
const commentForm = ref({ name: '', email: '', body: '' });
const commentSubmitted = ref(false);
const commentSubmitting = ref(false);

const sidebarOpen = computed(
  () => selectedPost.value || isCreating.value || isEditing.value
);

/* POSTS */
onMounted(loadPosts);

async function loadPosts() {
  postsLoading.value = true;
  try {
    const r = await fetch(
      `https://mate.academy/students-api/posts?userId=${USER_ID}`
    );
    posts.value = await r.json();
  } catch {
    postsError.value = true;
  } finally {
    postsLoading.value = false;
  }
}

function selectPost(p) {
  selectedPost.value = p;
  isCreating.value = false;
  isEditing.value = false;
  showCommentForm.value = false;
  loadComments(p.id);
}

async function loadComments(id) {
  commentsLoading.value = true;
  commentsError.value = false;
  comments.value = [];
  try {
    const r = await fetch(
      `https://mate.academy/students-api/comments?postId=${id}`
    );
    comments.value = await r.json();
  } catch {
    commentsError.value = true;
  } finally {
    commentsLoading.value = false;
  }
}

function openCreatePost() {
  isCreating.value = true;
  selectedPost.value = null;
  postForm.value = { title: '', body: '' };
}

async function savePost() {
  postFormSubmitted.value = true;
  if (!postForm.value.title || !postForm.value.body) return;

  if (isEditing.value) {
    Object.assign(selectedPost.value, postForm.value);
    isEditing.value = false;
  } else {
    const newPost = {
      ...postForm.value,
      id: Date.now(),
      userId: USER_ID,
    };
    posts.value.unshift(newPost);
    selectedPost.value = newPost;
    isCreating.value = false;
  }
}

function startEdit() {
  postForm.value = {
    title: selectedPost.value.title,
    body: selectedPost.value.body,
  };
  isEditing.value = true;
}

function deletePost() {
  posts.value = posts.value.filter(p => p !== selectedPost.value);
  closeSidebar();
}

function closeSidebar() {
  selectedPost.value = null;
  isCreating.value = false;
  isEditing.value = false;
}

/* COMMENTS */
async function submitComment() {
  commentSubmitted.value = true;
  if (!commentForm.value.name || !commentForm.value.email || !commentForm.value.body) return;

  commentSubmitting.value = true;
  try {
    const r = await fetch(
      `https://mate.academy/students-api/comments`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          ...commentForm.value,
          postId: selectedPost.value.id,
        }),
      }
    );
    const newComment = await r.json();
    comments.value.push(newComment);
    commentForm.value.body = '';
  } finally {
    commentSubmitting.value = false;
  }
}

function deleteComment(c) {
  comments.value = comments.value.filter(x => x !== c);
}

function clearCommentForm() {
  commentForm.value = { ...commentForm.value, body: '' };
  commentSubmitted.value = false;
}
</script>

<style>
.sidebar {
  position: fixed;
  top: 0;
  right: -420px;
  width: 420px;
  height: 100%;
  background: #1e1e1e;
  color: white;
  padding: 20px;
  transition: right 0.3s;
  overflow-y: auto;
}
.Sidebar--open {
  right: 0;
}
</style>
