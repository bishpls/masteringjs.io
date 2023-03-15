Type some code that uses Lodash in the below input, click "Run" (or press Shift+Enter) and see the results!
Clicking "Share" adds a base64 encoded version of your input to `window.location.hash`, so you can share your examples with other devs.

<style>
  button.opt {
    background-color: #19E2F1;
    color: white;
    padding: 0.5em;
    margin-right: 0.33em;
    border-radius: 4px;
    cursor: pointer;
    border: 0px;
    margin-bottom: 0.5em;
    font-size: 1em;
  }
</style>

<div>
<button class="opt" onclick="format()">&rsaquo; Run</button>
<button class="opt" onclick="tourl()">&Uarr; Share</button>
</div>
<div id="input" style="border: 1px solid #ddd"></div>

### Results:

<div id="output" style="border: 1px solid #ddd"></div>

<script src="../../codemirror-5.62.2/lib/codemirror.js"></script>
<script src="https://cdn.jsdelivr.net/npm/lodash@4.17.20/lodash.min.js"></script>
<link rel="stylesheet" href="../../codemirror-5.62.2/lib/codemirror.css">
<script src="../../codemirror-5.62.2/mode/javascript/javascript.js"></script>
<script type="text/javascript">
  const global = {cnsl: {} }
  const messages = [];
  const originalMethod = (global.cnsl.log = console.log);
  // https://glebbahmutov.com/blog/capture-all-the-logs/
  // https://www.bayanbennett.com/posts/how-does-mdn-intercept-console-log-devlog-003/
  console.log = function() {
      // messages.push(JSON.stringify(arguments[0])); // toString() causes [Object object]. No string conversion causes an error
      for (let i = 0; i < arguments.length; i++){
        const text = formatOutput(arguments[i]);
        messages.push(text);
      }
      const form = messages.join(" ");
      writeOutput(form);
      originalMethod.apply(console, arguments);
  }
  const input = CodeMirror(document.querySelector('#input'), {
    lineNumbers: true,
    tabSize: 2,
    value: window.location.hash ? atob(window.location.hash.slice(1)) : `console.log(_.omit({ a: 1, b: 2 }, ['a']));\n`,
    mode: 'javascript'
  });
  input.setOption('extraKeys', {
    'Shift-Enter': function() {
      format();
    }
  });
  const output = CodeMirror(document.querySelector('#output'), {
    lineNumbers: true,
    tabSize: 2,
    mode: 'javascript',
    readOnly: true
  });
  function tourl() {
    window.location.hash = btoa(input.getValue());
  }
  function format() {
    output.setValue('');
    const raw = input.getValue();
    const val = eval(raw);
    /*
    console.warn(messages);
    const res = messages.join('\n');
    messages.length = 0;
    const restore = () => {
      Object.keys(global.cnsl).forEach(methodName => {
        console[methodName] = global.cnsl[methodName]
      })
    }
    // restore();
    output.setValue(res);
    */
  }
  format();
  // https://github.com/mdn/bob/blob/main/editor/js/editor-libs/console-utils.js
  /**
 * Formats arrays:
 * - quotes around strings in arrays
 * - square brackets around arrays
 * - adds commas appropriately (with spacing)
 * designed to be used recursively
 * @param {any} input - The output to log.
 * @returns Formatted output as a string.
 */
function formatArray(input) {
  let output = "";
  for (let i = 0, l = input.length; i < l; i++) {
    if (typeof input[i] === "string") {
      output += '"' + input[i] + '"';
    } else if (Array.isArray(input[i])) {
      output += "Array [";
      output += formatArray(input[i]);
      output += "]";
    } else {
      output += formatOutput(input[i]);
    }
    if (i < input.length - 1) {
      output += ", ";
    }
  }
  return output;
}
/**
 * Formats objects:
 * ArrayBuffer, DataView, SharedArrayBuffer,
 * Int8Array, Int16Array, Int32Array,
 * Uint8Array, Uint16Array, Uint32Array,
 * Uint8ClampedArray, Float32Array, Float64Array
 * Symbol
 * @param {any} input - The output to log.
 * @returns Formatted output as a string.
 */
function formatObject(input) {
  "use strict";
  const bufferDataViewRegExp = /^(ArrayBuffer|SharedArrayBuffer|DataView)$/;
  const complexArrayRegExp =
    /^(Int8Array|Int16Array|Int32Array|Uint8Array|Uint16Array|Uint32Array|Uint8ClampedArray|Float32Array|Float64Array|BigInt64Array|BigUint64Array)$/;
  const objectName = input.constructor ? input.constructor.name : input;
  if (objectName === "String") {
    // String object
    return `String { "${input.valueOf()}" }`;
  }
  if (input === JSON) {
    // console.log(JSON) is outputed as "JSON {}" in browser console
    return `JSON {}`;
  }
  if (objectName.match && objectName.match(bufferDataViewRegExp)) {
    return objectName + " {}";
  }
  if (objectName.match && objectName.match(complexArrayRegExp)) {
    const arrayLength = input.length;
    if (arrayLength > 0) {
      return objectName + " [" + formatArray(input) + "]";
    } else {
      return objectName + " []";
    }
  }
  if (objectName === "Symbol" && input !== undefined) {
    return input.toString();
  }
  if (objectName === "Object") {
    let formattedChild = "";
    let start = true;
    for (const key in input) {
      if (Object.prototype.hasOwnProperty.call(input, key)) {
        if (start) {
          start = false;
        } else {
          formattedChild = formattedChild + ", ";
        }
        formattedChild = formattedChild + key + ": " + formatOutput(input[key]);
      }
    }
    return objectName + " { " + formattedChild + " }";
  }
  // Special object created with `OrdinaryObjectCreate(null)` returned by, for
  // example, named capture groups in https://mzl.la/2RERfQL
  // @see https://github.com/mdn/bob/issues/574#issuecomment-858213621
  if (!input.constructor && !input.prototype) {
    let formattedChild = "";
    let start = true;
    for (const key in input) {
      if (start) {
        start = false;
      } else {
        formattedChild = formattedChild + ", ";
      }
      formattedChild = formattedChild + key + ": " + formatOutput(input[key]);
    }
    return "Object { " + formattedChild + " }";
  }
  return input;
}
/**
 * Formats output to indicate its type:
 * - quotes around strings
 * - single quotes around strings containing double quotes
 * - square brackets around arrays
 * (also copes with arrays of arrays)
 * does NOT detect Int32Array etc
 * @param {any} input - The output to log.
 * @returns Formatted output as a string.
 */
function formatOutput(input) {
  if (input === undefined || input === null || typeof input === "boolean") {
    return String(input);
  } else if (typeof input === "number") {
    // Negative zero
    if (Object.is(input, -0)) {
      return "-0";
    }
    return String(input);
  } else if (typeof input === "bigint") {
    return String(input) + "n";
  } else if (typeof input === "string") {
    // string literal
    if (input.includes('"')) {
      return "'" + input + "'";
    } else {
      return '"' + input + '"';
    }
  } else if (Array.isArray(input)) {
    // check the contents of the array
    return "Array [" + formatArray(input) + "]";
  } else {
    return formatObject(input);
  }
}
/**
 * Writes the provided content to the editor’s output area
 * @param {String} content - The content to write to output
 */
// modify for codemirror
function writeOutput(content) {
  console.warn(content);
  const outputContent = output.getValue();
  const newLogItem = content + "\n";
  output.setValue(outputContent + newLogItem);
  messages.length = 0;
}
</script>