<script lang="ts">
  import CourseList from "./CourseList.svelte";
  import Links from "./Links.svelte";
  import TutorsInfo from "./TutorsInfo.svelte";
  import Welcome from "./Welcome.svelte";
</script>

<div class="to-accent-50 dark:to-accent-900 card m-4 border bg-gradient-to-l from-primary-50 dark:from-primary-900">
  <div class="container mx-auto mt-2 items-center justify-between lg:flex">
    <div class="mx-4">
      <Welcome />
    </div>
    <div class="mx-4 lg:w-1/2">
      <TutorsInfo />
      <Links />
    </div>
  </div>
</div>
<CourseList />
