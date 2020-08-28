<template>
  <v-dialog v-if="mounted" persistent max-width="40vw" v-model="form.enabled">
    <!-- Database connection form -->
    <v-card :disabled="form.loading && page == 0">
      <v-card-title>Database settings</v-card-title>
      <v-card-subtitle>{{form.values.status}}</v-card-subtitle>
      <div v-if="page == 0">
        <v-card-text class="mt-5 px-12">
          <v-form class="db-settings" ref="form" :value="test.value">
            <v-text-field
              :placeholder="form.uri_placeholder"
              :append-icon="test.icons[test.uri]"
              :rules="rules.uri"
              v-model="form.values.uri"
              label="URI"
              outlined
              required
            ></v-text-field>
            <v-text-field
              :rules="rules.credentials"
              v-model="form.values.username"
              :append-icon="test.icons[test.credentials]"
              placeholder="neo4j"
              label="Username"
              outlined
              required
            ></v-text-field>
            <v-text-field
              :type="form.password_masked ? 'password' : 'text'"
              :append-icon="test.icons[test.credentials]"
              :rules="rules.credentials"
              @focus="form.password_masked = false"
              @blur="form.password_masked = true"
              v-model="form.values.password"
              placeholder="****"
              label="Password"
              outlined
              required
            ></v-text-field>
          </v-form>
        </v-card-text>

        <v-card-actions class="px-5">
          <v-btn text :disabled="!db.populated" @click="db_settings_close()">Close</v-btn>
          <v-spacer></v-spacer>
          <v-btn outlined @click="db_settings_test()" :loading="test.busy">Test</v-btn>
          <v-btn outlined @click="db_settings_apply()" :loading="form.loading">Apply</v-btn>
        </v-card-actions>
      </div>

      <!-- Empty database helper -->
      <div v-else-if="page == 1">
        <v-card-title class="text-center ml-5">
          <v-icon size="36" class="warning--text px-5">mdi-hand-right</v-icon>Hold up, this database appears to be empty
        </v-card-title>
        <v-card-text color="grey lighten-5" class="px-12">
          <v-row>
            <v-col />
            <v-col cols="11">
              <div class="mb-2">
                <b>a) Either</b> run the ingestor to load an account of your own:
              </div>

              <v-card color="black" class="pa-2 white--text" style="font-size: 11px">
                [root@localhost ~]#
                <b>awspx ingest</b>
                <br />[-] The profile 'default' was not found. Would you like to create it? (y/n) y
                <br />AWS Access Key ID [None]: ****9XY7
                <br />AWS Secret Access Key [None]: ****ks91
                <br />Default region name [None]: eu-west-1
                <br />Default output format [None]: json
                <br />
              </v-card>
              <div class="mt-5 mb-2">
                <b>b) OR</b> load an existing database (e.g. the provided sample):
              </div>
              <v-card color="black" class="pa-2 white--text" style="font-size: 11px">
                [root@localhost ~]#
                <b>awspx db --load-zip sample.zip</b>
                <br />[*] Importing records from /opt/awspx/data/sample.zip
                <br />
                <br />[root@localhost ~]#
                <b>awspx attacks</b>
                <br />[*] Searching database for attack patterns
                <br />
              </v-card>
            </v-col>
            <v-col />
          </v-row>
        </v-card-text>
        <v-card-actions>
          <v-btn text @click="page = 0">Back</v-btn>
          <v-spacer></v-spacer>
          <v-btn outlined @click="db_settings_apply()" :loading="form.loading">Check Again</v-btn>
        </v-card-actions>
      </div>
    </v-card>
  </v-dialog>
</template>

<script>
export default {
  name: "Database",
  data: function() {
    return {
      page: 0,
      form: {
        enabled: true,
        values: {
          uri: `bolt://${new URL(location.href).host}:7687`,
          username: this.neo4j.auth.username,
          password: this.neo4j.auth.password,
          status: "Disconnected"
        },
        uri_placeholder: `bolt://${new URL(location.href).host}:7687`,
        password_masked: true,
        loading: false
      },
      db: {
        connected: false,
        populated: false
      },
      test: {
        busy: false,
        value: false,
        icons: {
          null: "",
          true: "mdi-check-circle-outline",
          false: "mdi-alert-circle-outline"
        },
        uri: null,
        credentials: null
      },
      mounted: false
    };
  },

  methods: {
    form_reset_validation() {
      this.test = {
        ...this.test,
        uri: null,
        credentials: null
      };
    },

    db_settings_open() {
      if (!this.form.enabled) this.form.enabled = true;
    },

    db_settings_close() {
      this.form.enabled = false;
    },

    db_settings_test() {
      this.neo4j.error = {};
      this.test.busy = true;

      const test = {
        uri: null,
        credentials: null
      };

      return this.neo4j
        .test(
          this.form.values.uri,
          this.form.values.username,
          this.form.values.password
        )
        .then(r => {
          test.uri = true;
          test.credentials = true;

          return true;
        })
        .catch(e => {
          if (this.mounted) {
            switch (e.code) {
              case "Neo.ClientError.Security.Unauthorized":
              case "Neo.ClientError.Security.AuthenticationRateLimit":
                test.credentials = false;
                break;
              default:
                test.uri = false;
                break;
            }
            this.neo4j.error = e;
          }
          return false;
        })
        .finally(() => {
          this.test.busy = false;
          this.test = {
            ...this.test,
            ...test
          };

          if (typeof this.$refs.form !== "undefined")
            this.$refs.form.validate();
        });
    },

    db_settings_apply() {
      localStorage.DB = JSON.stringify({
        uri: this.form.values.uri,
        username: this.form.values.username,
        password: this.form.values.password
      });

      this.neo4j.setup(
        this.form.values.uri,
        this.form.values.username,
        this.form.values.password
      );

      this.initialize();
    },

    initialize() {
      const types = ["Admin", "External", "Resource", "CatchAll", "Generic"];
      let resources = [];
      let actions = [];

      this.form.loading = true;

      Promise.all([
        this.neo4j
          .run(
            "MATCH ()-[action:ACTION]->() " +
              "WITH DISTINCT {" +
              "    Name: action.Name, " +
              "    Description: action.Description, " +
              "    Access: action.Access, " +
              "    Reference: action.Reference" +
              "} AS action RETURN action " +
              "ORDER BY action.Name",
            false
          )
          .then(results => {
            actions = results.Text.map(result => result["action"]);
          }),
        this.neo4j
          .run(
            "MATCH (r) WHERE NOT (r:Pattern OR r:`AWS::Domain`) RETURN r",
            false
          )
          .then(elements => {
            resources = elements.Graph.map(r => {
              const classification = r.classes
                .filter(c => types.indexOf(c) != -1)
                .concat("")[0];
              const id =
                typeof r.data.properties.Arn !== "undefined"
                  ? r.data.properties.Arn
                  : r.data.name;

              return {
                name: r.data.name,
                id: id,
                type: r.data.name === "Effective Admin" ? "Admin" : r.data.type,
                class:
                  r.data.name === "Effective Admin" ? "Admin" : classification,
                element: r
              };
            }).sort((a, b) => {
              let c = types.indexOf(a.class) - types.indexOf(b.class);
              if (c !== 0) return c;
              else return a.name.toLowerCase() < b.name.toLowerCase() ? -1 : 1;
              return c;
            });
          })
      ])
        .then(() => {
          this.db.connected = true;
          this.db.populated = actions.concat(resources).length > 0;
          this.form.values.status = `Connected to ${
            this.neo4j.auth.uri.split("/")[2]
          }`;

          this.$emit("resources", resources);
          this.$emit("actions", actions);
        })
        .catch(e => {
          this.db.connected = false;
          this.form.values.status = "Disconnected";
        })
        .finally(() => {
          let page = 0;
          this.form.loading = false;

          if (this.db.connected && this.db.populated) this.form.enabled = false;
          else if (this.db.connected) page = 1;

          this.page = page;
        });
    },

    autologin() {
      if ("DB" in localStorage) {
        const auth = JSON.parse(localStorage.DB);
        this.form.values.uri = auth.uri;
        this.form.values.username = auth.username;
        this.form.values.password = auth.password;
      }

      this.mounted = false;

      this.db_settings_test()
        .then(success => {
          if (success) {
            this.neo4j.setup(
              this.form.values.uri,
              this.form.values.username,
              this.form.values.password
            );
            this.initialize();
          }
        })
        .finally(() => {
          this.mounted = true;
        });
    }
  },
  watch: {
    "form.values.uri"() {
      this.form_reset_validation();
    },
    "form.values.username"() {
      this.form_reset_validation();
    },
    "form.values.password"() {
      this.form_reset_validation();
    }
  },
  computed: {
    rules() {
      const rules = {
        uri: [this.test.uri === null || this.test.uri],
        credentials: [this.test.credentials === null || this.test.credentials]
      };

      return rules;
    }
  },

  mounted() {
    // this.mounted = true;
    this.autologin();
  }
};
</script>


<style>
.db-settings .mdi-check-circle-outline::before {
  color: #4caf50;
}
.db-settings .mdi-alert-circle-outline::before {
  color: #ff5252;
}
</style>