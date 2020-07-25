<template>
  <v-container
    id="course-map"
    fluid
    tag="section"
  >
    <v-row>
      <v-col cols="12">
        <base-material-card
          color="primary"
          :title="'Course Map: ' + courseMapData.Meta.title"
          class="px-5 py-3 text-h3"
          max-height="1000"
          max-width="1400"
        >
          <v-card-text class="px-0 pb-0">
            <div ref="main">
              <v-sheet height="600">
                <v-btn
                  v-for="course in courseMapData.Curriculum"
                  :key="course.id"
                  small
                  :color="getColorForStatus(courseStatus[course.id])"
                  :style="{ left: course.view.x + '%', top: course.view.y + '%' }"
                  class="inline-block course-btn"
                  @click="handleClick(courseMapData, courseStatus, course.id)"
                >
                  <v-tooltip bottom>
                    <template v-slot:activator="{ on, attrs }">
                      <span
                        v-bind="attrs"
                        v-on="on"
                      >
                        {{ course.information.dept }} {{ course.information.coursenum }}
                      </span>
                    </template>
                    <span>{{ course.information.title }}</span>
                  </v-tooltip>
                </v-btn>
                <v-stage :config="konfig.canvasSize">
                  <v-layer>
                    <div
                      v-for="rect in konfig.rect"
                      :key="'Rectange_x_' + rect.x + '_y_' + rect.y + '_w_' + rect.width + '_h_' + rect.height"
                    >
                      <v-rect
                        :config="rect"
                      />
                    </div>
                  </v-layer>
                </v-stage>
              </v-sheet>
            </div>
          </v-card-text>
        </base-material-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script>

  import CSB2023 from './../../../../data/map/CSB2023.json'
  import CURRICULUM from './../../../../data/db/CURRICULUM.json'

  const getMergedData = (cur, map) => {
    const data = {
      Meta: map.banner,
      Curriculum: cur,
      Lines: map.lines,
    }
    for (let i = 0; i < data.Curriculum.length; i++) {
      for (let j = 0; j < map.display.length; j++) {
        if (data.Curriculum[i].id === map.display[j].id) {
          data.Curriculum[i].view = map.display[j].view
        }
      }
    }
    data.Curriculum = data.Curriculum.filter(e => e.view !== undefined)
    return data
  }

  const getPrereqsByIndex = (d, i) => {
    const catalogYear = d.Meta.catalog
    return d.Curriculum[i].information.prereqs[catalogYear]
  }

  const getPrereqsById = (d, id) => {
    for (let i = 0; i < d.Curriculum.length; i++) {
      if (d.Curriculum[i].id === id) {
        return getPrereqsByIndex(d, i)
      }
    }
    return []
  }

  const getPostreqsById = (d, id) => {
    const postreqs = []
    for (let i = 0; i < d.Curriculum.length; i++) {
      const prereqs = getPrereqsByIndex(d, i).flat()
      if (prereqs.indexOf(id) !== -1) {
        postreqs.push(d.Curriculum[i].id)
      }
    }
    return postreqs
  }

  const getInitStatus = (d) => {
    const ret = {}
    for (let i = 0; i < d.Curriculum.length; i++) {
      if (getPrereqsByIndex(d, i).length === 0) {
        ret[d.Curriculum[i].id] = 'Available'
      } else {
        ret[d.Curriculum[i].id] = 'Unavailable'
      }
    }
    return ret
  }

  const getRect = (canvasSize, line) => {
    return {
      x: line.x * canvasSize.width / 100,
      y: line.y * canvasSize.height / 100,
      width: line.w * canvasSize.width / 100,
      height: line.h * canvasSize.height / 100,
      stroke: line.opt ? line.opt : 'black',
      strokeWidth: 4,
    }
  }

  const getRectangles = (canvasSize, lines) => {
    const init = []
    for (let i = 0; i < lines.length; i++) {
      init.push(getRect(canvasSize, lines[i]))
    }
    return init
  }

  const data = getMergedData(CURRICULUM, CSB2023)
  const status = getInitStatus(data)

  const canvasSize = {
    height: 600,
    width: 1000,
  }

  const konfig = {
    canvasSize: canvasSize,
    rect: getRectangles(canvasSize, data.Lines),
  }

  export default {
    data: () => ({
      courseStatus: status,
      courseMapData: data,
      konfig: konfig,
    }),
    mounted () {
      window.addEventListener('resize', this.onResize)
      this.onResize()
    },
    destroyed () {
      window.removeEventListener('resize', this.onResize)
    },
    methods: {
      getColorForStatus: (status) => {
        if (status === 'Unavailable') { return '' }
        if (status === 'Completed') { return 'accent' }
        return 'primary'
      },
      handleClick: function (data, status, id) {
        if (status[id] === 'Unavailable' || status[id] === 'Available') {
          this.handleDownChain(data, status, id)
        } else if (status[id] === 'Completed') {
          this.handleUpChain(data, status, id)
        }
        this.checkAvailability(data, status)
      },
      handleDownChain: function (data, status, id) {
        if (status[id]) {
          status[id] = 'Completed'
        }
        const prereqs = getPrereqsById(data, id)
        for (let i = 0; i < prereqs.length; i++) {
          if (typeof prereqs[i] === 'string') {
            this.handleDownChain(data, status, prereqs[i])
          } else {
            prereqs[i].forEach(c => {
              this.handleDownChain(data, status, c)
            })
          }
        }
      },
      handleUpChain: function (data, status, id) {
        if (status[id]) {
          status[id] = 'Unavailable'
        }
        const postreqs = getPostreqsById(data, id)
        for (let i = 0; i < postreqs.length; i++) {
          this.handleUpChain(data, status, postreqs[i])
        }
      },
      checkAvailability: function (d, status) {
        for (let i = 0; i < d.Curriculum.length; i++) {
          if (status[d.Curriculum[i].id] === 'Unavailable') {
            let available = true
            const prereqs = getPrereqsByIndex(d, i)
            for (let j = 0; j < prereqs.length; j++) {
              if (typeof prereqs[j] === 'string') {
                if (status[prereqs[j]] !== 'Completed') {
                  available = false
                }
              } else { // Nested Prereq
                let oneComp = false
                prereqs[j].forEach(c => {
                  if (status[c] === 'Completed') {
                    oneComp = true
                  }
                })
                if (oneComp === false) {
                  available = false
                }
              }
            }
            if (available === true) {
              status[d.Curriculum[i].id] = 'Available'
            }
          }
        }
      },
      setup (sketch) {
        sketch.background('green')
        sketch.text('Hello p5!', 20, 20)
      },
      onResize () {
        const vm = this
        const width = vm.$refs.main.clientWidth
        const height = vm.$refs.main.clientHeight
        vm.$data.konfig.canvasSize.width = width
        vm.$data.konfig.canvasSize.height = height
        vm.$data.konfig.rect = []
        vm.$data.konfig.rect = getRectangles(vm.$data.konfig.canvasSize, vm.$data.courseMapData.Lines)
      },
    },
  }
</script>

<style scoped lang="scss">
  .course-btn {
    width: 5%;
    margin: 0.5% !important;
    padding: 0 !important;
    position: absolute;
  }

  .course-btn.isUnavailable {
    background-color: gray !important;
    color: black !important;
  }

</style>
