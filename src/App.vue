<template>
  <div id="PageContainer">
    <svg id="drawing" width="100%" height="100%" viewBox="0 0 500 500"></svg>
    <center>
      <input
        type="range"
        min="0"
        :max="inputPrj.max_val"
        value="0"
        class="slider"
        id="myRange"
        v-model="slide_val"
        v-if="inputPrj.control_type=='slider'"
      >
      <div id="output" v-if="inputPrj.display_value">{{slide_val}}</div>
      <div class="btn_container">
        <div
          v-for="(n,index) in inputPrj.button_labels"
          :key="n"
          class="slide_btn"
          :class="{'activeBtn':button==n}"
          @click="change_face(index+1)"
          v-if="inputPrj.control_type=='buttons'"
        >{{n}}</div>
      </div>
    </center>
  </div>
</template>
<script>
import SVG from "svg.js";
import { close } from "fs";
export default {
  data() {
    return {
      inputPrj,
      SVGinput,
      slide_val: 0,
      intervals: 4,
      canvas: "",
      button: "",
      previous_val: 0,
      animate_buttons_step: 10,
      input_fronts: {},
      input_numbers: {},
      path_objects: {},
      was_slided: 0
    };
  },
  watch: {
    slide_val(value) {
      if (this.was_slided) $(".mrNext").removeClass("disabled");
      this.was_slided++;
      this.animate(value);
      $(".mrEdit").val(value);
    }
  },

  mounted() {
    $(".mrNext").addClass("disabled");
    this.slide_val = this.inputPrj.max_val / 2;
    this.canvas = SVG("drawing").viewbox(0, 0, 500, 500);
    for (var i = 0; i < Object.keys(this.SVGinput.input_path).length; i++) {
      //drawing path shapes
      if (this.SVGinput.input_type[i] == "path") {
        this.path_objects[i] = this.canvas
          .path(this.SVGinput.input_path[i][0])
          .attr({
            linecap: "round",
            "stroke-linejoin": "null",
            "stroke-dasharray": "null",
            "stroke-width":
              this.SVGinput.input_border[i].split("*")[0] != ""
                ? this.SVGinput.input_border[i].split("*")[0]
                : 0,
            stroke:
              this.SVGinput.input_border[i].split("*")[1] != ""
                ? this.SVGinput.input_border[i].split("*")[1]
                : "black",
            fill: "none",
            id: "object" + i
          });
      }
      if (this.SVGinput.input_type[i] == "circle") {
        this.path_objects[i] = this.canvas
          .circle(this.SVGinput.input_path[i][0].split(",")[0])
          .attr({
            "stroke-width":
              this.SVGinput.input_border[i].split("*")[0] != ""
                ? this.SVGinput.input_border[i].split("*")[0]
                : 0,
            stroke:
              this.SVGinput.input_border[i].split("*")[1] != ""
                ? this.SVGinput.input_border[i].split("*")[1]
                : "black",
            cx: this.SVGinput.input_path[i][0].split(",")[1],
            cy: this.SVGinput.input_path[i][0].split(",")[2],
            id: "object" + i
          });
      }

      if (this.SVGinput.input_type[i] == "ellipse") {
        this.path_objects[i] = this.canvas
          .ellipse(
            this.SVGinput.input_path[i][0].split(",")[0],
            this.SVGinput.input_path[i][0].split(",")[1]
          )
          .attr({
            "stroke-width":
              this.SVGinput.input_border[i].split("*")[0] != ""
                ? this.SVGinput.input_border[i].split("*")[0]
                : 0,
            stroke:
              this.SVGinput.input_border[i].split("*")[1] != ""
                ? this.SVGinput.input_border[i].split("*")[1]
                : "black",
            cx: this.SVGinput.input_path[i][0].split(",")[2],
            cy: this.SVGinput.input_path[i][0].split(",")[3],
            id: "object" + i
          });
      }
    }

    // this.path_objects[99] = this.canvas.circle(50);

    // this.path_objects[99].attr({
    //   r: 30,
    //   cx: 0,
    //   cy: 0
    // });

    this.add_colors();
    this.create_splits();

    this.rotate_all(this.slide_val);
    this.animate(this.slide_val);
  },
  methods: {
    change_face(x) {
      var vue_OBJ = this;
      var temp;
      var min;
      this.button = x;
      this.slide_val = (this.inputPrj.max_val / this.intervals) * (x - 1);

      // console.log("in change", this.previous_val, this.slide_val);
      this.animate(this.previous_val);

      temp = this.previous_val;
      min = this.slide_val;

      if (this.previous_val < this.slide_val) {
        temp = this.previous_val;
        min = this.slide_val;
        for (var i = temp; i <= this.slide_val; i++) {
          setTimeout(function() {
            vue_OBJ.animate(temp);
            temp++;
          }, (min - i) * vue_OBJ.animate_buttons_step);
        }
      } else {
        temp = this.previous_val;
        min = this.previous_val;
        for (var i = temp; i >= this.slide_val; i--) {
          setTimeout(function() {
            vue_OBJ.animate(temp);
            temp--;
          }, (min - i) * vue_OBJ.animate_buttons_step);
        }
      }

      // console.log('button',x);
    },
    animate(x) {
      // console.log("in animate for:", x, this.previous_val);
      var int_size = this.inputPrj.max_val / this.intervals;
      var int_number = 0;
      var int_poz = parseInt(x);
      while (int_poz > int_size) {
        int_poz = int_poz - int_size;
        int_number++;
      }

      for (var i = 0; i < Object.keys(this.SVGinput.input_path).length; i++) {
        if (this.SVGinput.input_type[i] == "path") {
          this.path_objects[i].plot(
            this.recalculate_path(i, int_number, int_poz)
          );
        }
        if (this.SVGinput.input_type[i] == "circle") {
          this.path_objects[i].attr({
            r:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[0]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[0],
                  this.SVGinput.input_path[i][int_number].split(",")[0],
                  int_poz
                )
              ),
            cx:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[1]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[1],
                  this.SVGinput.input_path[i][int_number].split(",")[1],
                  int_poz
                )
              ),
            cy:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[2]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[2],
                  this.SVGinput.input_path[i][int_number].split(",")[2],
                  int_poz
                )
              )
          });
        }
        if (this.SVGinput.input_type[i] == "ellipse") {
          this.path_objects[i].attr({
            rx:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[0]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[0],
                  this.SVGinput.input_path[i][int_number].split(",")[0],
                  int_poz
                )
              ),
            ry:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[1]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[1],
                  this.SVGinput.input_path[i][int_number].split(",")[1],
                  int_poz
                )
              ),
            cx:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[2]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[2],
                  this.SVGinput.input_path[i][int_number].split(",")[2],
                  int_poz
                )
              ),
            cy:
              parseInt(this.SVGinput.input_path[i][int_number].split(",")[3]) +
              parseInt(
                this.calc_difference(
                  this.SVGinput.input_path[i][int_number + 1].split(",")[3],
                  this.SVGinput.input_path[i][int_number].split(",")[3],
                  int_poz
                )
              )
          });
        }
      }

      this.rotate_all(this.slide_val);
      this.previous_val = x;
    },
    calc_difference(val1, val2, int_poz) {
      return (
        ((val1 - val2) / (this.inputPrj.max_val / this.intervals)) * int_poz
      );
    },
    recalculate_path(i, int_number, int_poz) {
      var rez = "";
      var temp = [];
      for (
        var elem = 0;
        elem < this.input_numbers[i][int_number].length;
        elem++
      ) {
        temp[elem] =
          this.input_numbers[i][int_number][elem] +
          this.calc_difference(
            this.input_numbers[i][int_number + 1][elem],
            this.input_numbers[i][int_number][elem],
            int_poz
          );

        rez = rez + this.input_fronts[i][int_number][elem] + temp[elem];
      }
      if (
        this.input_numbers[i][int_number].length <
        this.input_fronts[i][int_number].length
      ) {
        rez =
          rez +
          this.input_fronts[i][int_number][
            this.input_numbers[i][int_number].length
          ];
      }

      return rez;
    },
    rotate_all(x) {
      for (var i = 0; i < Object.keys(this.SVGinput.input_path).length; i++) {
        if (this.SVGinput.input_rotate[i]) {
          var element = document.getElementById("object" + i);
          var temp_val = parseInt(this.inputPrj.max_val);
          do {
            temp_val = temp_val / 10;
          } while (temp_val > 100);
          element.setAttribute(
            "transform",
            "rotate(" + (this.inputPrj.max_val / 2 - x) / temp_val + ",250,0) "
          );
        }
      }
    },
    add_colors() {
      for (var i = 0; i < Object.keys(this.SVGinput.input_path).length; i++) {
        var vue_OBJ = this;
        switch (this.SVGinput.input_colors[i].type) {
          case "simple":
            this.path_objects[i].fill(this.SVGinput.input_colors[i]);
            break;
          case "linear":
            var linear_color = this.canvas.gradient("linear", function(stop) {
              var color_array = vue_OBJ.SVGinput.input_colors[i].color.split(
                "*"
              );
              var stops_array = vue_OBJ.SVGinput.input_colors[i].stops.split(
                "*"
              );
              // console.log(color_array);
              for (var j = 0; j < color_array.length; j++) {
                stop.at(parseFloat(stops_array[j]), color_array[j]);
              }
            });
            if (
              vue_OBJ.SVGinput.input_colors[i].from &&
              vue_OBJ.SVGinput.input_colors[i].to
            ) {
              // console.log(vue_OBJ.SVGinput.input_colors[i]);
              var from1 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].from.split(",")[0]
              );
              var from2 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].from.split(",")[1]
              );
              var to1 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].to.split(",")[0]
              );
              var to2 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].to.split(",")[1]
              );

              linear_color.from(from1, from2).to(to1, to2);
            }

            this.path_objects[i].fill(linear_color);
            break;
          case "radial":
            var radial = this.canvas.gradient("radial", function(stop) {
              var color_array = vue_OBJ.SVGinput.input_colors[i].color.split(
                "*"
              );
              var stops_array = vue_OBJ.SVGinput.input_colors[i].stops.split(
                "*"
              );
              // console.log(color_array);
              for (var j = 0; j < color_array.length; j++) {
                // console.log(parseFloat(stops_array[j]), color_array[j]);
                stop.at(parseFloat(stops_array[j]), color_array[j]);
              }
            });

            if (
              vue_OBJ.SVGinput.input_colors[i].from &&
              vue_OBJ.SVGinput.input_colors[i].to
            ) {
              // console.log(vue_OBJ.SVGinput.input_colors[i]);
              var from1 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].from.split(",")[0]
              );
              var from2 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].from.split(",")[1]
              );
              var to1 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].to.split(",")[0]
              );
              var to2 = parseFloat(
                vue_OBJ.SVGinput.input_colors[i].to.split(",")[1]
              );

              radial.from(from1, from2).to(to1, to2);
            }
            if (vue_OBJ.SVGinput.input_colors[i].radius) {
              radial.radius(vue_OBJ.SVGinput.input_colors[i].radius);
            }

            this.path_objects[i].fill(radial);
            break;
          default:
            // console.log(i, "other type");
            break;
        }
      }
    },
    create_splits() {
      for (
        var collection = 0;
        collection < Object.keys(this.SVGinput.input_path).length;
        collection++
      ) {
        // console.log("collection:", collection);
        this.input_fronts[collection] = [];
        this.input_numbers[collection] = [];
        for (
          var indx = 0;
          indx < this.SVGinput.input_path[collection].length;
          indx++
        ) {
          this.input_fronts[collection].push(
            this.extract_fronts(this.SVGinput.input_path[collection][indx])
          );
          this.input_numbers[collection].push(
            this.extract_numbers(this.SVGinput.input_path[collection][indx])
          );
        }
      }
      // console.log(this.SVGinput.input_path, this.input_fronts);
      // console.log( this.SVGinput.input_path[0], this.input_fronts[0], this.input_numbers[0]
      // );
    },
    extract_fronts(sir) {
      var rez = sir;
      rez = this.findAndReplace(rez, "0", "");
      rez = this.findAndReplace(rez, "1", "");
      rez = this.findAndReplace(rez, "2", "");
      rez = this.findAndReplace(rez, "3", "");
      rez = this.findAndReplace(rez, "4", "");
      rez = this.findAndReplace(rez, "5", "");
      rez = this.findAndReplace(rez, "6", "");
      rez = this.findAndReplace(rez, "7", "");
      rez = this.findAndReplace(rez, "8", "");
      rez = this.findAndReplace(rez, "9", "");
      rez = this.findAndReplace(rez, ".", "");
      rez = this.findAndReplace(rez, "-", "");
      return rez;
    },
    extract_numbers(sir) {
      var rez = sir;
      rez = this.findAndReplace(rez, "z", "");
      rez = this.findAndReplace(rez, "Z", "");
      rez = this.findAndReplace(rez, "m", "");
      rez = this.findAndReplace(rez, "M", "");
      rez = this.findAndReplace(rez, "c", "#");
      rez = this.findAndReplace(rez, "C", "#");
      rez = this.findAndReplace(rez, " ", "#");
      rez = this.findAndReplace(rez, ",", "#");
      rez = rez.split("#");
      for (var i = 0; i < rez.length; i++) {
        rez[i] = parseFloat(rez[i]);
      }
      return rez;
    },
    findAndReplace(string, target, replacement) {
      var i = 0,
        length = string.length;
      for (i; i < length; i++) {
        string = string.replace(target, replacement);
      }
      return string;
    }
  }
};
</script>
<style scoped>
#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
}

@media only screen and (max-width: 400) {
  #PageContainer {
    width: 100%;
    height: auto;
    max-height: 60vh;
  }
}

/* #PageContainer {
  height: 300px;
} */
#output {
  width: 60px;
  height: 50px;
  font-size: xx-large;
}
/* for slider */

#myRange {
  height: 50px;
}
input[type="range"] {
  /*removes default webkit styles*/
  -webkit-appearance: none;

  /*fix for FF unable to apply focus style bug */
  border: 1px solid white;

  /*required for proper track sizing in FF*/
  max-width: 300px;
}
input[type="range"]::-webkit-slider-runnable-track {
  max-width: 300px;
  height: 5px;
  background: #ddd;
  border: none;
  border-radius: 3px;
}
input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  border: none;
  height: 16px;
  width: 16px;
  border-radius: 50%;
  background: goldenrod;
  margin-top: -4px;
}
input[type="range"]:focus {
  outline: none;
}
input[type="range"]:focus::-webkit-slider-runnable-track {
  background: #ccc;
}

input[type="range"]::-moz-range-track {
  width: 300px;
  height: 5px;
  background: #ddd;
  border: none;
  border-radius: 3px;
}
input[type="range"]::-moz-range-thumb {
  border: none;
  height: 16px;
  width: 16px;
  border-radius: 50%;
  background: goldenrod;
}

/*hide the outline behind the border*/
input[type="range"]:-moz-focusring {
  outline: 1px solid white;
  outline-offset: -1px;
}

input[type="range"]::-ms-track {
  width: 300px;
  height: 5px;

  /*remove bg colour from the track, we'll use ms-fill-lower and ms-fill-upper instead */
  background: transparent;

  /*leave room for the larger thumb to overflow with a transparent border */
  border-color: transparent;
  border-width: 6px 0;

  /*remove default tick marks*/
  color: transparent;
}
input[type="range"]::-ms-fill-lower {
  background: #777;
  border-radius: 10px;
}
input[type="range"]::-ms-fill-upper {
  background: #ddd;
  border-radius: 10px;
}
input[type="range"]::-ms-thumb {
  border: none;
  height: 16px;
  width: 16px;
  border-radius: 50%;
  background: goldenrod;
}
input[type="range"]:focus::-ms-fill-lower {
  background: #888;
}
input[type="range"]:focus::-ms-fill-upper {
  background: #ccc;
}

.slide_btn {
  background: cadetblue;
  padding: 0 5px;
  height: 30px;
  min-width: 50px;
  display: inline-block;
  border: 1px solid;
  cursor: pointer;
  line-height: 30px;
  text-decoration: none;
  /* padding: 0 16px; */
  color: #fff;
  text-align: center;
  letter-spacing: 0.5px;
  transition: background-color 0.2s ease-out;
  -webkit-tap-highlight-color: transparent;
  vertical-align: middle;
  box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.14),
    0 3px 1px -2px rgba(0, 0, 0, 0.12), 0 1px 5px 0 rgba(0, 0, 0, 0.2);
  box-sizing: inherit;
}
.activeBtn {
  background: #086d64;
  transform: scale(1.2);
}
.btn_container {
  display: flex;
  align-items: center;
  justify-content: center;
  max-height: 60px;
}
#drawing {
  /* min-height: 30vh; */
  max-height: 40vh;
}
</style>
<style>
label {
  width: 100%;
}
.question {
  padding-bottom: 10px;
}
.mrEdit {
  display: none !important;
}
</style>  