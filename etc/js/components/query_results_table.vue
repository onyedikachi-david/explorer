<script>
  module.exports = {
    name: "query-results-table",
    props: {
      columns: {type: Object, required: true},
      show_this: {type: Boolean, required: true},
      column_style: {type: Array, required: false },
      row_style: {type: Function, required: false },
      row_order: {type: Object, required: false, default: { kind: 'this', mode: 'asc', index: 0 } }
    },
    data: function() {
      return {
        order_by: { 
          kind: this.row_order.kind, 
          mode: this.row_order.mode,
          index: this.row_order.index,
          mode_index: this.mode == 'asc' ? 1 : (this.mode == 'desc' ? 2 : 3)
        },
        group_enabled: {},
        columnWidths: [],
        resizing: { active: false, index: -1, startX: 0, startWidth: 0 },
        resizeObserver: null
      }
    },
    mounted() {
      console.log('[QueryTable] Component mounted');
      // Initialize column widths
      this.initColumnWidths();
      
      // Add window event listeners for resizing
      window.addEventListener('mousemove', this.handleResize, { passive: false });
      window.addEventListener('mouseup', this.stopResize);

      // Handle parent container resizing
      this.resizeObserver = new ResizeObserver(() => {
        if (!this.resizing.active) {
          this.adjustColumnWidths();
        }
      });
      this.resizeObserver.observe(this.$el.closest('.content-container') || this.$el);

      // Store initial column widths
      this.$nextTick(() => {
        const headers = this.$el.querySelectorAll('th');
        let initialWidths = [];
        headers.forEach((header, index) => {
          if (!header.classList.contains('query-results-squeeze')) {
            this.$set(this.columnWidths, index, header.offsetWidth);
            initialWidths.push(header.offsetWidth);
          }
        });
        console.log('[QueryTable] Initial column widths:', initialWidths);
      });
    },
    beforeDestroy() {
      console.log('[QueryTable] Component unmounting, removing event listeners');
      window.removeEventListener('mousemove', this.handleResize);
      window.removeEventListener('mouseup', this.stopResize);
      if (this.resizeObserver) {
        this.resizeObserver.disconnect();
      }
    },
    computed: {
      var_count: function() {
        if (this.columns.vars) {
          return this.columns.vars.length;
        }
        return 0;
      }
    },
    methods: {
      // Get sorted index for i
      index(i) {
        if (this.columns.data.index && this.columns.data.index.length != 0) {
          return this.columns.data.index[i].index;
        } else {
          return i;
        }
      },
      // Get type info for term
      type_info(term) {
        let column = this.var_count + term;
        if (this.column_style && this.column_style[column].type) {
          return this.column_style[column].type;
        } else if (this.columns.type_info) {
          let id = this.columns.ids[term];
          if (Array.isArray(id)) {
            let result = this.columns.type_info[id[0]];
            if (result) {
              return result;
            }
            return this.columns.type_info[id[1]];
          } else {
            return this.columns.type_info[id];
          }
        }
      },
      // Format header of (component) term
      id_elem(str) {
        const elems = str.split('.');
        return elems[elems.length - 1];
      },
      term_header(term) {
        const id = this.columns.ids[term];
        let result;

        if (Array.isArray(id)) {
          /* New format, pairs are split up in array */
          if (id.length == 1) {
            result = this.id_elem(id[0]);
          } else {
            const first = this.id_elem(id[0]);
            const second = this.id_elem(id[1]);
            result = "(" + first + "," + second + ")";
          }
        } else {
          /* Old format, backwards compatibility */
          const pair = id.split(',');
          if (pair.length == 1) {
            result = this.id_elem(pair[0]);
          } else {
            const first = this.id_elem(pair[0].slice(1));
            const second = this.id_elem(pair[1].slice(0, -1));
            result = "(" + first + "," + second + ")";
          }
        }

        let ti = this.type_info(term);
        if (ti) {
          // If type info has a single member we can use its type info for the
          // entire column, otherwise ignore.
          const keys = Object.keys(ti);
          if (keys.length == 1) {
            ti = ti[keys[0]];
          } else {
            ti = undefined;
          }
        }
        if (ti) {
          // Extended type properties are in the second element of the array
          const ext = ti[1];
          if (ext && ext.symbol) {
            if (ext.quantity != "flecs.units.Duration") {
              result += "\xa0(" + ext.symbol + ")";
            }
          }
        }
        return result;
      },
      // Is term a component or a tag
      term_is_tag(term) {
        if (!this.columns.data.values) {
          return true;
        }
        return !this.columns.data.values[term];
      },
      group_is_enabled(group) {
        let result = this.group_enabled[group];
        if (result === undefined) {
          result = true;
        }
        return result;
      },
      toggle_group(group) {
        let enabled = this.group_is_enabled(group);
        this.group_enabled[group] = !enabled;
      },
      create_none(h) {
        return h('td', [h('span', { class: "query-result-cell-none" }, ["None"])]);
      },
      // Order by logic
      on_order_by(evt) {
        let modes;
        let mode_index = 0;

        if (evt.kind !== 'var') {
          modes = ['asc', 'desc'];
        } else {
          modes = ['asc', 'desc', 'group', 'group_only'];
        }

        if (evt.kind == this.order_by.kind && evt.index == this.order_by.index) {
          mode_index = this.order_by.mode_index;
          mode_index = (mode_index + 1) % modes.length;
        }

        this.order_by.kind = evt.kind;
        this.order_by.index = evt.index;
        this.order_by.mode_index = mode_index;
        this.order_by.mode = modes[mode_index];

        this.$emit('order-by', this.order_by);
      },
      header_title(index, value) {
        if (this.column_style && this.column_style[index]) {
          return this.column_style[index].name;
        }
        return value;
      },
      header_style(index) {
        if (this.column_style && this.column_style[index]) {
          return this.column_style[index].style;
        }
      },
      getColumnStyle(index) {
        const width = this.columnWidths[index] || 150;
        return {
          width: `${width}px`,
          minWidth: '50px',
          maxWidth: 'none',
          position: 'relative',
          overflow: 'hidden',
          whiteSpace: 'nowrap',
          textOverflow: 'ellipsis',
          boxSizing: 'border-box'
        };
      },
      initColumnWidths() {
        // Count total columns including entity column and value columns
        let totalColumns = 0;
        if (this.show_this) totalColumns++;
        if (this.columns.vars) totalColumns += this.columns.vars.length;
        if (this.columns.ids) {
          totalColumns += this.columns.ids.filter(id => !this.term_is_tag(this.columns.ids.indexOf(id))).length;
        }
        this.columnWidths = new Array(totalColumns).fill(150);
        console.log('[QueryTable] Initialized column widths:', { totalColumns, widths: this.columnWidths });
      },
      startResize(event, index) {
        console.log('[QueryTable] Starting resize:', { index, event: event.type });
        event.preventDefault();
        event.stopPropagation();
        
        // Force stop any ongoing resize
        if (this.resizing.active) {
          this.stopResize();
        }
        
        const target = event.target;
        const th = target.closest('th');
        if (!th) return;
        
        const initialWidth = th.offsetWidth;
        const initialX = event.pageX;
        
        this.resizing = {
          active: true,
          index,
          startX: initialX,
          startWidth: initialWidth,
          element: th
        };
        
        // Add visual feedback
        document.body.style.cursor = 'col-resize';
        target.classList.add('resizing');
        
        // Force style update
        this.$nextTick(() => {
          if (th) {
            th.style.width = `${initialWidth}px`;
            th.style.minWidth = '50px';
            th.style.maxWidth = 'none';
          }
        });
      },
      handleResize(event) {
        if (!this.resizing.active) return;
        
        event.preventDefault();
        event.stopPropagation();
        
        const deltaX = event.pageX - this.resizing.startX;
        const newWidth = Math.max(50, this.resizing.startWidth + deltaX);
        
        // Update the column width in our state
        this.$set(this.columnWidths, this.resizing.index, newWidth);
        
        // Update the header cell width
        if (this.resizing.element) {
          this.resizing.element.style.width = `${newWidth}px`;
          
          // Find and update all cells in this column
          const table = this.resizing.element.closest('table');
          if (table) {
            const cells = table.querySelectorAll(`td:nth-child(${this.resizing.index + 1})`);
            cells.forEach(cell => {
              cell.style.width = `${newWidth}px`;
              cell.style.minWidth = '50px';
              cell.style.maxWidth = 'none';
            });
          }
        }
      },
      stopResize() {
        if (!this.resizing.active) return;
        
        document.body.style.cursor = '';
        if (this.resizing.element) {
          this.resizing.element.classList.remove('resizing');
        }
        
        this.resizing = { active: false, index: -1, startX: 0, startWidth: 0, element: null };
      },
      resetColumnWidth(index) {
        this.$set(this.columnWidths, index, 150);
      },
      // Create table header
      create_header(h) {
        console.log('[QueryTable] Creating header with column widths:', this.columnWidths);
        let ths = [];
        const data = this.columns.data;
        const columns = this.columns;

        const order_by_icon = {
          asc: "codicons:triangle-down",
          desc: "codicons:triangle-up",
          group: "codicons:diff-added",
          group_only: "codicons:diff-removed"
        }[this.order_by.mode];

        const icon = h('icon', { 
          props: { icon: order_by_icon, size: 10 } 
        });

        const icon_elem = h('span', { 
          class: 'query-results-order-by-icon' }, 
        [icon]);

        const icon_placeholder = h('span', {
          class: 'query-results-order-by-icon-placeholder'
        }, ["\xa0"]);

        if (this.order_by.mode !== 'group_only') {
          let column = 0;

          if (this.row_style) {
            // Insert placeholder in header for row icon
            ths.push(h('th', {}));
          }

          if (data.entities && data.entities.length && this.show_this) {
            const th = h('th', { 
              style: this.getColumnStyle(column),
              on: {
                click: () => { this.on_order_by({kind: 'this'}); }
              }
            }, [
              this.header_title(column, "Entity"),
              this.order_by.kind === "this" ? icon_elem : icon_placeholder,
              h('div', {
                class: 'column-resizer',
                on: {
                  mousedown: (e) => { 
                    e.stopPropagation();
                    this.startResize(e, column);
                  },
                  dblclick: (e) => {
                    e.stopPropagation();
                    this.resetColumnWidth(column);
                  }
                }
              })
            ]);
            ths.push(th);
            column++;
          }

          if (columns.vars) {
            let i = 0;
            for (let var_name of columns.vars) {
              var_name_elems = var_name.split(".");
              var_name = var_name_elems[var_name_elems.length - 1];
              var_name = this.header_title(column, var_name);

              let index = i;
              const th = h('th', { 
                style: this.getColumnStyle(column),
                on: {
                  click: () => { this.on_order_by({kind: 'var', index: index}); }
                }
              }, [
                var_name,
                (this.order_by.kind === 'var' && this.order_by.index === i) ? icon_elem : icon_placeholder,
                h('div', {
                  class: 'column-resizer',
                  on: {
                    mousedown: (e) => { 
                      e.stopPropagation(); 
                      this.startResize(e, index);
                    },
                    dblclick: (e) => {
                      e.stopPropagation();
                      this.resetColumnWidth(index);
                    }
                  }
                })
              ]);
              ths.push(th);
              column++; i++;
            }
          }

          if (columns.ids) {
            for (let i = 0; i < columns.ids.length; i ++) {
              if (!this.term_is_tag(i)) {
                let name = this.term_header(i);
                name = this.header_title(column, name);

                let index = i;
                const th = h('th', { 
                  style: this.getColumnStyle(column),
                  on: {
                    click: () => { this.on_order_by({kind: 'value', index: index}); }
                  }
                }, [
                  name,
                  (this.order_by.kind === 'value' && this.order_by.index === i) ? icon_elem : icon_placeholder,
                  h('div', {
                    class: 'column-resizer',
                    on: {
                      mousedown: (e) => { 
                        e.stopPropagation(); 
                        this.startResize(e, column);
                      },
                      dblclick: (e) => {
                        e.stopPropagation();
                        this.resetColumnWidth(column);
                      }
                    }
                  })
                ]);
                ths.push(th);
                column++;
              }
            }
          }

          ths.push(h('th', { class: 'query-results-squeeze' }));
          ths.push(h('th', {}));
        }

        return h('thead', 
          { class: ['query-results-table-header'] },
          [ h('tr', { class: 'query-results-header-row' }, ths) ]
        );
      },
      // Create entity table cells
      create_entities(h, entities, labels) {
        let td_entities = [];
        if (!entities) return td_entities;

        for (let i = 0; i < entities.length; i++) {
          const index = this.index(i);
          const entity = entities[index];
          const label = labels[index];

          if (entity === '*') {
            td_entities.push(this.create_none(h));
            continue;
          }

          const hierarchy = h('entity-hierarchy', {
            props: { entity_path: entity }
          });

          const ref = h('entity-reference', {
            props: {
              entity: entity,
              label: label,
              show_name: true,
              show_parent: false
            },
            on: this.$listeners
          });

          td_entities.push(h('td', {
            style: this.getColumnStyle(0)
          }, [hierarchy, ref]));
        }

        return td_entities;
      },
      create_row_icons(h, count) {
        let td_icons = [];
        for (let i = 0; i < count; i ++) {
          const index = this.index(i);
          const style = this.row_style(this.columns, index);
          let icon = h('icon', {
            props: { icon: style.icon, size: 18, opacity: 0.7 }
          });
          td_icons.push(h('td', {class: 'query-results-row-icon'}, [icon]));
        }
        return td_icons;
      },
      // Create alert table cells
      create_alerts(h, alerts) {
        let td_alerts = [];
        for (let i = 0; i < alerts.length; i ++) {
          const index = this.index(i);
          const alert = alerts[index];
          let icon = undefined;

          if (alert === true) {
            icon = h('icon', {
              props: { icon: "feather:alert-triangle", size: 18, opacity: 0.7 }
            });
          }

          td_alerts.push(h('td', {class: 'query-results-row-icon'}, [icon]));
        }
        return td_alerts;
      },
      // Create variable table cells
      create_vars(h) {
        const columns = this.columns;
        const data = this.columns.data;
        let vars = [];
        let var_groups = [];

        if (columns.vars) {
          for (let i = 0; i < columns.vars.length; i ++) {
            vars.push(
              this.create_entities(h, data.vars[i], data.var_labels[i])
            );
          }

          // Find var groups
          if (this.order_by.mode === 'group' || this.order_by.mode === 'group_only') {
            const var_values = data.vars[this.order_by.index];
            let var_last = undefined;
            let i = 0;

            for (let el of this.columns.data.index) {
              const var_cur = el.order_by;
              if ((var_cur !== var_last)) {
                if (var_groups.length) {
                  var_groups[var_groups.length - 1].count = 
                    i - var_groups[var_groups.length - 1].row;
                }
                var_groups.push({
                  group: var_cur,
                  label: el.label,
                  row: i,
                });
                var_last = var_cur;
              }
              i ++;
            }
            var_groups[var_groups.length - 1].count =
              i - var_groups[var_groups.length - 1].row;
          }
        }

        return [vars, var_groups];
      },
      // Create component value table cells
      create_values(h) {
        const columns = this.columns;
        const data = this.columns.data;
        let values = [];

        let columnIndex = this.show_this ? 1 : 0;
        if (this.columns.vars) {
          columnIndex += this.columns.vars.length;
        }

        for (let i = 0; i < columns.ids.length; i ++) {
          if (this.term_is_tag(i)) {
            continue;
          }

          let value_array = [];
          if (data.values[i]) {
            for (let v = 0; v < data.values[i].length; v ++) {
              const index = this.index(v);
              if (!data.is_set[i][index]) {
                value_array.push(this.create_none(h));
              } else {
                const inspector = h('inspector-props', {
                  props: {
                    value: data.values[i][index],
                    type: this.type_info(i),
                    list: true
                  },
                  on: this.$listeners 
                });

                value_array.push(h('td', {
                  style: this.getColumnStyle(columnIndex)
                }, [inspector]));
              }
            }
          }
          values.push(value_array);
          columnIndex++;
        }

        return values;
      },
      // Create table row
      create_row(h, children, color, row_class, key) {
        let left_border_color = color;
        let row_color = color;
        let class_name = "query-results-row";
        if (row_class) {
          class_name += " " + row_class;
        }

        if (!color) {
          left_border_color = "var(--steel-700)";
          row_color = "var(--steel-500)";
        }
        
        let style = "border-left-color: " + left_border_color + "; ";
        style += "background-color: " + row_color;

        return h('tr', { class: class_name, style: style, attrs: {key: key} }, children );
      },
      group_cell(h, group, label, count, colspan) {
        const tree = h('entity-hierarchy', {
          props: { entity_path: group }});

        let chevron = h('icon', {
          props: { 
            icon: this.group_is_enabled(group) 
              ? 'codicons:chevron-down' 
              : 'codicons:chevron-right',
            opacity: 0.7,
            top: 1,
          }});

        chevron = h('div', {
          attrs: {
            style: 'float: left; margin-right: 5px;'
          }
        }, [chevron]);

        if (this.order_by.mode === 'group_only') {
          chevron = undefined;
        }

        const el = h('div', {
        }, [tree, label + "\xa0(" + count + ")"]);

        return h('td', {
          class: 'query-results-group noselect',
          attrs: {
            colspan: colspan + 1,
          }
        }, [chevron, el]);
      },
      // Create table body
      create_body(h) {
        const columns = this.columns;
        const data = columns.data;
        let column_count = 0;
        let trs = [];
        const [vars, var_groups] = this.create_vars(h);

        if (this.order_by.mode !== 'group_only') {
          // Create cells from columns
          let tds = {
            entities: [],
            vars: vars,
            values: this.create_values(h),
            alerts: this.create_alerts(h, data.alerts),
            row_icons: []
          };

          if (this.show_this) {
            tds.entities = this.create_entities(h, data.entities, data.labels);
          }
          if (this.row_style) {
            tds.row_icons = this.create_row_icons(h, data.entities.length);
          }

          // Initialize rows
          let rows = [];
          for (let i = 0; i < columns.count; i ++) {
            rows.push([]);
          }

          // Populate row children arrays with td elements

          // Add row icons
          if (this.row_style) {
            for (let i = 0; i < tds.row_icons.length; i ++) {
              rows[i].push(tds.row_icons[i]);
            }
          }

          // Add entity tds
          if (tds.entities.length) {
            column_count ++;
          }
          for (let i = 0; i < tds.entities.length; i ++) {
            rows[i].push(tds.entities[i]);
          }

          // Add variable tds
          for (let var_tds of tds.vars) {
            for (let i = 0; i < var_tds.length; i ++) {
              rows[i].push(var_tds[i]);
            }
            column_count ++;
          }

          // Add value tds
          for (let value_tds of tds.values) {
            for (let i = 0; i < value_tds.length; i ++) {
              rows[i].push(value_tds[i]);
            }
            column_count ++;
          }

          // Add td's at the end that push values to the left
          for (let row of rows) {
            row.push( h('td', { class: 'query-results-squeeze'}) );
          }

          // Add alert tds
          for (let i = 0; i < tds.alerts.length; i ++) {
            rows[i].push(tds.alerts[i]);
          }

          // Create row elements
          for (let i = 0; i < rows.length; i ++) {
            let color, row_class, key, index = this.index(i);
            if (data.colors) {
              color = data.colors[index];
            }
            if (this.row_style) {
              let style = this.row_style(this.columns, index);
              if (!color) {
                color = style.background_color;
              }
              row_class = style.class;
            }
            if (data.entities) {
              key = data.entities[index];
            }
            trs.push(this.create_row(h, rows[i], color, row_class, key));
          }

          // If variable grouping is enabled, insert group headers
          if (this.order_by.mode === 'group') {
            let inserted = 0;
            for (let group of var_groups) {
              const group_td = this.group_cell(
                h, group.group, group.label, group.count, column_count);

              let group_tr_sep = h('tr', {
                class: 'query-results-group-separator'
                }, [h('td', { attrs: { colspan: column_count + 1 } })]);

              let group_class = 'query-results-group-row';
              if (this.group_is_enabled(group.group)) {
                group_class += ' query-results-group-row-enabled';
              }

              let group_tr = h('tr', { 
                class: group_class,
                on: {
                  click: (evt) => {
                    this.toggle_group(group.group);
                    this.$forceUpdate();
                  }
                }
              }, [group_td]);

              trs.splice(group.row + inserted, 0, [group_tr_sep, group_tr]);
              inserted ++;

              // Remove group rows if group is disabled
              if (!this.group_is_enabled(group.group)) {
                trs.splice(group.row + inserted, group.count);
                inserted -= group.count;
              }
            }
          }
        } else {
          for (let group of var_groups) {
            const group_td = this.group_cell(
              h, group.group, group.label, group.count, column_count);

            let group_tr = h('tr', { 
              class: 'query-results-group-row' 
            }, [group_td]);

            trs.push(group_tr);
          }
        }

        return h('tbody', trs);
      },
      reset() {
        this.order_by = { kind: 'this', mode: 'asc', mode_index: 0 };
        this.group_enabled = {};
        this.$emit('order-by', this.order_by);
      },
      adjustColumnWidths() {
        const containerWidth = this.$el.offsetWidth;
        const totalCurrentWidth = this.columnWidths.reduce((sum, width) => sum + width, 0);
        
        if (totalCurrentWidth > 0) {
          const ratio = containerWidth / totalCurrentWidth;
          this.columnWidths.forEach((width, index) => {
            const newWidth = Math.max(50, Math.floor(width * ratio));
            this.$set(this.columnWidths, index, newWidth);
          });
        }
      }
    },
    render: function(h) {
      const header = this.create_header(h);
      const body = this.create_body(h);
      return h('table', {class: 'query-results-table'}, [header, body]);
    }
  };
</script>

<style>
  div.query-results-stats {
    display: flex;
    position: absolute;
    justify-content: flex-start;
    padding: 10px;
    color: var(--secondary-text);
  }

  div.query-results-nav {
    display: flex;
    justify-content: flex-end;
    background-color: var(--panel-footer-bg);
    padding-top: 3px;
    padding-bottom: 3px;
    padding-right: 6px;
    border-style: solid;
    border-width: 1px;
    border-left-width: 0px;
    border-right-width: 0px;
    border-color: var(--grey-900);
  }

  div.query-results-nav span {
    padding: 5px;
    color: var(--tertiary-text);
  }

  div.query-results-nav input {
    border: none;
    background-color: var(--panel-bg-secondary);
    color: var(--secondary-text);
    width: 40px;
    border-style: solid;
    border-width: 1px;
    border-color: var(--panel-bg-secondary);
    border-radius: 4px;
    margin-right: 10px;
  }

  div.query-results-nav input:focus {
    border-color: var(--steel-400);
  }

  div.query-results-nav button {
    padding: 5px;
    padding-left: 7px;
    padding-right: 7px;
    border: none;
    cursor: pointer;
    background-color: var(--panel-footer-bg);
    color: var(--secondary-text);
  }

  div.query-results-nav button:hover  {
    background-color: var(--steel-650);
    color: var(--primary-text);
  }

  div.query-results-nav button:active  {
    background-color: var(--steel-600);
    color: var(--primary-text);
  }

  div.query-results-nav button:disabled  {
    background-color: var(--panel-footer-bg);
    color: var(--tertiary-text);
    cursor: default;
  }

  td.query-results-row-icon img {
    margin-top: 5px; /* center icon in row */
  }

  div.query-result-yesno {
    color: var(--secondary-text);
    padding-top: 10px;
    padding-bottom: 10px;
    padding-left: 10px;
  }

  span.query-result-cell-none {
    color: var(--tertiary-text);
  }

  span.query-result-count {
    font-weight: 300;
    opacity: 0.8;
  }

  table.query-results-table {
    width: 100%;
    border-collapse: collapse;
    table-layout: fixed !important;
    position: relative;
    z-index: 1;
  }

  table.query-results-table th,
  table.query-results-table td {
    position: relative;
    padding: 8px;
    border: 1px solid var(--grey-900);
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    box-sizing: border-box;
    min-width: 50px;
  }

  table.query-results-table th:first-child {
    text-align: left;
  }

  table.query-results-table th {
    padding-left: var(--p-4);
    padding-right: var(--p-4);
    padding-bottom: var(--p-4);
    padding-top: var(--p-4);
    font-weight: 400;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    -o-user-select: none;
    user-select: none;
    color: var(--label);
    cursor: pointer;
    position: relative;
    background-color: transparent;
    transition: background-color 0.1s ease-in-out;
  }

  table.query-results-table th:hover {
    background-color: var(--steel-700);
  }

  th.query-results-squeeze:hover {
    background-color: transparent !important;
    cursor: default !important;
  }

  table.query-results-table thead.query-results-table-header {
    background-color: var(--row-bg);
    position: sticky;
    top: 0px;
    z-index: 2;
    box-shadow: 0px 2px 10px rgba(0, 0, 0, 0.35);
  }

  table.query-results-table th {
    white-space: nowrap;
  }

  tr.query-results-group-row {
    border-style: solid;
    border-width: 0px;
    border-left-width: 3px;
    border-color: var(--steel-700);
    position: relative;
  }

  tr.query-results-group-row-enabled {
    box-shadow: 0px 4px 8px rgba(0, 0, 0, 0.35);
  }

  tr.query-results-group-separator td {
    height: 3px !important;
    background-color: var(--steel-800) !important;
  }

  td.query-results-group {
    color: var(--secondary-text);
    background-color: var(--steel-800) !important;
    cursor: pointer;
  }

  span.query-results-order-by-icon {
    position: relative;
    opacity: 0.5;
    margin-left: 5px;
  }

  span.query-results-order-by-icon-placeholder {
    margin-left: 5px;
    width: 10px;
    height: 10px;
  }

  table.query-results-table thead tr,
  table.query-results-table tbody tr {
    background-color: var(--row-bg);
    transition: background-color 0.2s ease;
    transition: border-color 0.2s ease;
  }

  table.query-results-table tbody tr td {
    background-color: var(--cell-bg);
    transition: background-color 0.2s ease;
    transition: border-color 0.2s ease;
  }

  table.query-results-table tbody img {
    transition: opacity 0.2s ease;
  }

  table.query-results-table span {
    transition: color 0.1s ease;
  }

  table.query-results-table tbody tr:hover td {
    background-color: var(--cell-bg-hover);
  }

  tr.query-results-row {
    border-style: solid;
    border-width: 0px;
    border-left-width: 3px;
    border-bottom-width: 1.5px;
    border-bottom-color: var(--row-bg);
  }

  tr.query-results-header-row {
    position: relative;
    left: -2px;
  }

  th.query-results-squeeze,
  td.query-results-squeeze {
    width: auto !important;
    min-width: 0 !important;
    max-width: none !important;
    padding: 0 !important;
    margin: 0 !important;
  }

  span.query-result-color {
    width: 10px;
    max-width: 10px;
  }

  table.query-results-table td {
    padding-left: var(--p-4);
    padding-right: var(--p-4);
    padding-bottom: 0px;
    padding-top: 0px;
    border-top-color: var(--grey-900);
    border-top-style: solid;
    border-top-width: 1.5px;
    font-size: inherit;
    font-feature-settings: "tnum";
    height: 46px;
    background-color: var(--cell-bg);
    transition: background-color 0.2s ease;
  }

  .column-resizer {
    position: absolute;
    top: 0;
    right: -3px;
    width: 6px;
    height: 100%;
    cursor: col-resize;
    user-select: none;
    background: transparent;
    z-index: 100;
    touch-action: none;
  }

  .column-resizer:hover,
  .column-resizer.active {
    background-color: var(--primary-color);
    opacity: 0.3;
  }

  th.resizing {
    background-color: var(--steel-700) !important;
    user-select: none;
  }

  th.resizing .column-resizer {
    background-color: var(--primary-color);
    opacity: 0.5;
    width: 2px;
    right: -1px;
  }

  /* Ensure content doesn't affect column width */
  table.query-results-table th > *:not(.column-resizer),
  table.query-results-table td > * {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    width: 100%;
    pointer-events: auto;
  }

  /* Prevent text selection while resizing */
  .resizing * {
    user-select: none !important;
    pointer-events: none !important;
  }

</style>
