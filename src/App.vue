<template>
  <v-app>
    <v-app-bar app flat color="primary" dark>
      <div class="d-flex align-center display-1 font-weight-bold">
        <svg
          style="height: 2.5rem; width: 2.5rem"
          class="mr-2"
          fill="none"
          stroke="currentColor"
          viewBox="0 0 24 24"
          xmlns="http://www.w3.org/2000/svg"
        >
          <path
            stroke-linecap="round"
            stroke-linejoin="round"
            stroke-width="2.2"
            d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253"
          ></path>
        </svg>
        Kindle 2 PDF
      </div>
      <v-spacer />
      <v-btn
        class="sm-none"
        href="https://twitter.com/@dakmig"
        depressed
        color="blue"
        >Follow on Twitter</v-btn
      >
    </v-app-bar>

    <v-main>
      <v-snackbar
        v-model="snack.show"
        :timeout="snack.timeout"
        :color="snack.color"
        shaped
        top
        left
      >
        {{ snack.message }}
        <template v-slot:action="{ attrs }">
          <v-btn text v-bind="attrs" @click="snack.show = false"> Close </v-btn>
        </template>
      </v-snackbar>
      <v-dialog
        v-model="dialog"
        fullscreen
        hide-overlay
        transition="dialog-bottom-transition"
      >
        <v-card>
          <v-toolbar dark color="primary">
            <v-btn icon dark @click="dialog = false">
              <svg
                style="height: 2rem; width: 2rem"
                fill="none"
                stroke="currentColor"
                viewBox="0 0 24 24"
                xmlns="http://www.w3.org/2000/svg"
              >
                <path
                  stroke-linecap="round"
                  stroke-linejoin="round"
                  stroke-width="2.5"
                  d="M6 18L18 6M6 6l12 12"
                ></path>
              </svg>
            </v-btn>
            <v-toolbar-title>Generate PDF</v-toolbar-title>
            <v-spacer></v-spacer>
            <v-toolbar-items>
              <v-btn dark text @click="savePDF"> Save PDF </v-btn>
            </v-toolbar-items>
          </v-toolbar>
          <v-card-text ref="pdfArea">
            <div v-for="book in books" :key="book.title" class="mx-4">
              <h2 class="book-heading font-weight-bold mt-4 black--text">
                {{ book.title }}
              </h2>
              <p
                class="book-p my-4"
                v-for="(value, idx) in book.values"
                :key="idx"
              >
                <strong class="black--text">{{ idx + 1 }}</strong
                >. {{ value }}
              </p>
            </div>
          </v-card-text>
        </v-card>
      </v-dialog>

      <v-container
        class="d-flex flex-column justify-center align-center fill-height"
      >
        <h2 class="mb-6 font-weight-bold text-center">
          The easiest and most secure way to convert your kindle highlights to
          PDF!
        </h2>
        <v-sheet
          color="white"
          elevation="2"
          height="400"
          width="500"
          class="pa-2"
          max-width="100%"
          shaped
        >
          <file-upload
            :drop="true"
            :drop-directory="false"
            ref="upload"
            v-model="files"
            @input-filter="inputFilter"
            class="fill-height fill-width d-flex align-center justify-center font-weight-bold title-1 flex-column"
            :disabled="files.length > 0"
          >
            <svg
              class="mb-4"
              :class="{
                'primary--text': files.length === 0,
                'success--text': files.length > 0,
              }"
              style="height: 3rem; width: 3rem"
              fill="none"
              stroke="currentColor"
              viewBox="0 0 24 24"
              xmlns="http://www.w3.org/2000/svg"
            >
              <path
                stroke-linecap="round"
                stroke-linejoin="round"
                stroke-width="2.3"
                d="M4 16v1a3 3 0 003 3h10a3 3 0 003-3v-1m-4-8l-4-4m0 0L8 8m4-4v12"
              ></path>
            </svg>

            <p v-if="files.length === 0">
              Drop or Click to upload clippings.txt
            </p>
            <div v-else>
              <p>Uploaded {{ files[0].file.name }} Successfully!</p>
            </div>
          </file-upload>
        </v-sheet>
        <div class="text-center">
          <v-btn
            class="mt-4 mr-4"
            color="success"
            :disabled="files.length === 0"
            @click="parseKindle"
            >Convert</v-btn
          >
          <v-btn
            class="mt-4 mr-4"
            color="error"
            @click="$refs.upload.clear()"
            :disabled="files.length === 0"
            >Clear</v-btn
          >
          <v-btn
            @click="
              openSnack(
                'Connect kindle to your Laptop / PC and find the My Clippings.txt file in Documents',
                'warning',
                -1
              )
            "
            target="_blank"
            color="warning"
            class="mt-4"
            >Where can I find Clippings?</v-btn
          >
        </div>
      </v-container>
    </v-main>
    <v-footer padless>
      <v-col class="text-center" cols="12">
        {{ new Date().getFullYear() }} — Built by
        <strong> <a href="https://twitter.com/@dakmig">Daksh</a> </strong>,
        Creator of <strong><a href="https://nasch.io">Nasch</a></strong>
      </v-col>
    </v-footer>
  </v-app>
</template>

<script>
const END_OF_LINE = "\n";
const KINDLE_NOTE_DELIMITER = "==========";

// rewrote the parser orignally by https://elvisciotti.github.io/kindleparser/
function parseMyClippings(inputContent) {
  // read into map of sets
  let organisedNotesMap = new Map();
  inputContent.split(KINDLE_NOTE_DELIMITER).forEach((note) => {
    let lines = note.trim().split(END_OF_LINE);
    let title = lines[0].trim();
    if (title.length > 1) {
      let body = lines.slice(2).join(END_OF_LINE);
      if (!organisedNotesMap.has(title)) {
        organisedNotesMap.set(title, new Set());
      }
      let bodyTrimmed = trimNewLinesSpacesDots(body);
      bodyTrimmed = bodyTrimmed.charAt(0).toUpperCase() + bodyTrimmed.slice(1);
      organisedNotesMap.get(title).add(bodyTrimmed);
    }
  });

  const notesArray = Array.from(organisedNotesMap);
  const books = [];
  for (let i = 0; i < notesArray.length; i++) {
    const [title, texts] = notesArray[i];

    const temp = Array.from(Array.from(texts).values());
    const newNotes = new Set();

    temp.forEach((item) => {
      let highest = item;
      temp.forEach((item2) => {
        if (item2.indexOf(highest) > -1) {
          if (highest.length < item2.length) {
            highest = item2;
          }
        }
      });
      newNotes.add(highest);
    });

    books.push({ title, values: Array.from(newNotes.values()) });
  }
  return books;
}

function trimNewLinesSpacesDots(content) {
  return (
    content
      // starting
      .replace(/([\.\r\n]+)$/, "")
      .replace(/([\.\n]+)$/, "")
      // ending
      .replace(/(^[\.\r\n]+)/, "")
      .replace(/(^[\.\n]+)/, "")
      .trim()
  );
}
export default {
  name: "App",

  components: {},

  data: () => ({
    dialog: false,
    files: [],
    snack: {
      show: false,
      timeout: -1,
      color: "",
      message: "",
    },
    books: [],
  }),
  methods: {
    savePDF() {
      const el = this.$refs.pdfArea;
      this.openSnack("Saving PDF. Please wait!", "success", 5000);
      try {
        window
          .html2pdf()
          .from(el)
          .set({
            margin: 1,
            html2canvas: { scale: 1 },
            pagebreak: { mode: ["avoid-all", "css", "legacy"] },

            jsPDF: { unit: "in", format: "letter", orientation: "portrait" },
            filename: "kindle-clips.pdf",
          })
          .save();
      } catch (e) {
        console.log(e);
        this.openSnack(
          "Sorry failed to save! Please try again!",
          "error",
          5000
        );
      }
    },
    async parseKindle() {
      const file = this.files[0].file;
      const text = await file.text();

      this.books = parseMyClippings(text).sort(
        (a, b) => b.values.length - a.values.length
      );
      this.dialog = true;
    },
    openSnack(message, color = "primary", timeout = -1) {
      this.snack.message = message;
      this.snack.timeout = timeout;
      this.snack.color = color;
      this.snack.show = true;
    },
    inputFilter(newFile, oldFile, prevent) {
      if (newFile && !oldFile) {
        // Filter non-txt file
        if (!/\.(txt)$/i.test(newFile.name)) {
          this.openSnack("Please upload a .txt file only!", "error");
          return prevent();
        }
      }

      // Create a blob field
      newFile.blob = "";
      let URL = window.URL || window.webkitURL;
      if (URL && URL.createObjectURL) {
        newFile.blob = URL.createObjectURL(newFile.file);
      }
    },
  },
  watch: {
    "files.length"(val) {
      if (val === 0) {
        this.openSnack("Successfully cleared the file!", "success", 10000);
      } else {
        this.openSnack("Successfully uploaded the file!", "success", 1000);
      }
    },
  },
};
</script>


<style>
.fill-width {
  width: 100% !important;
}
.title-1 {
  font-size: 1.5rem;
}

.book-heading {
  font-size: 1.8rem !important;
  line-height: 2.6rem;
  font-weight: 900;
}
.book-p {
  font-size: 1.3rem;
  font-weight: 500;
  line-height: 1.95rem;
  margin-bottom: 0.75rem;
}

@media (max-width: 600px) {
  .sm-none {
    display: none !important;
  }
}
</style>