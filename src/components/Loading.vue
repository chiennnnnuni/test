<template>
  <div class="container">
    <div class="bars">
      <div class="bar bounce-min"></div>
      <div class="bar bounce"></div>
      <div class="bar bounce"></div>
      <div class="bar bounce"></div>
      <div class="bar bounce-min"></div>
    </div>
    <div class="text">- ㅎㅁㅎ -</div>
  </div>
</template>

<style>
.container {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(4, 30, 66, 0.7);
  z-index: 10;
}

.text {
  margin-top: 50px;
  color: #fda0d7;
  font-size: 22px;
}

.bars {
  display: flex;
  gap: 8px;
}

.bar {
  width: 15px;
  height: 30px;
  background-color: #fda0d7;
}

.bounce{
  animation: bounce 1s infinite ease-in-out;
}

.bounce-min{
  animation: bounce-min 1s infinite ease-in-out;
}

.bounce:nth-child(2){
  animation-delay: 0.2s;
}

.bounce:nth-child(3), .bounce-min:nth-child(1) {
  animation-delay: 0.5s;
}

.bounce:nth-child(4) {
  animation-delay: 0.9s;
  
}

@keyframes bounce {
  0%, 100% {
    transform: scaleY(0.5);
  }
  50% {
    transform: scaleY(2.5);
  }
}

@keyframes bounce-min {
  0%, 100% {
    transform: scaleY(0.7);
  }
  50% {
    transform: scaleY(1.2);
  }
}
</style>