<template>
  <div 
    class="clip-card" 
    :class="{ active: isActive }" 
    @click="$emit('select')"
  >
    <div class="thumbnail">
      <img :src="clip.thumbnail" :alt="clip.title">
      <span class="duration">{{ clip.duration }}</span>
    </div>
    <div class="info">
      <h4 class="title">{{ clip.title }}</h4>
      <p class="author">{{ clip.author }}</p>
    </div>
  </div>
</template>

<script>
export default {
  props: {
    clip: Object,
    isActive: Boolean
  }
}
</script>

<style scoped>
.clip-card {
  display: flex;
  gap: 12px;
  padding: 10px;
  border-radius: var(--radius);
  cursor: pointer;
  transition: var(--transition);
  margin-bottom: 8px;
  border: 1px solid transparent;
}

.clip-card:hover {
  background: var(--glass);
  transform: translateX(4px);
}

.clip-card.active {
  background: var(--glass);
  border-color: var(--accent);
}

.thumbnail {
  position: relative;
  width: 100px;
  height: 56px;
  flex-shrink: 0;
  border-radius: 6px;
  overflow: hidden;
  background: #000;
}

.thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  transition: var(--transition);
}

.clip-card:hover .thumbnail img {
  transform: scale(1.1);
}

.duration {
  position: absolute;
  bottom: 4px;
  right: 4px;
  background: rgba(0,0,0,0.8);
  font-size: 10px;
  padding: 2px 4px;
  border-radius: 3px;
  color: #fff;
}

.info {
  display: flex;
  flex-direction: column;
  justify-content: center;
}

.title {
  font-size: 13px;
  font-weight: 600;
  color: var(--text-primary);
  margin-bottom: 2px;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
  overflow: hidden;
}

.author {
  font-size: 11px;
  color: var(--text-secondary);
}
</style>
