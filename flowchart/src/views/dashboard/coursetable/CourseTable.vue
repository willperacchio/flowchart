<template>
  <v-container
    id="course-table"
    fluid
    tag="section"
  >
    <base-v-component
      heading="Course Table"
      link="components/simple-tables"
    />

    <base-material-card
      icon="mdi-clipboard-text"
      title="Course Table"
      class="px-5 py-3"
    >
      <v-data-table>
        <thead>
          <tr>
            <th class="primary--text">
              Course ID
            </th>
            <th class="primary--text">
              Title
            </th>
            <th class="primary--text">
              Credits
            </th>
            <th class="primary--text">
              Description
            </th>
            <th class="primary--text">
              Notes
            </th>
          </tr>
        </thead>

        <tbody>
          <tr
            v-for="course in curriculum"
            :key="course"
          >
            <td>{{ course.id }}</td>
            <td>{{ course.information.title }}</td>
            <td>{{ course.information.credits }}</td>
            <td>{{ course.information.coursedesc }}</td>
            <td>{{ course.information.coursenotes }}</td>
          </tr>
        </tbody>
      </v-data-table>
    </base-material-card>

    <div class="py-3" />
  </v-container>
</template>

<script>
  import curriculum from './../../../../data/db/CURRICULUM.json'

  export default {
    data: () => ({
      curriculum: curriculum,
    }),
  }
</script>
