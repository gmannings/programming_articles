# Refactoring for unit testing and new functionality

There are times when code enters branches after poor or non-existent pull-requests and
peer review, or has been included because it 'just works' and a feature needs to be shipped.
This sort of code is technical debt, it will potentially cause problems in the future,
and often isn't unit tested (often because it's very difficult to unit test).

The mechanisms for preventing complex or difficult to support/test code are varied, and include:

* Linting
* Static code analysis tools
* Pull-requests/peer review
* Enforcing coding standards

## An example of less than ideal code

This code has a number of bad practices, can you tell what it does?

```html
<!DOCTYPE html>
<html>
<head>
    <title>Not ideal code</title>
</head>
<body>
    <div id="containerDiv">
        <div class="groupDiv">
            <select class="mainSel">
                <option value="cat1">Category 1</option>
                <option value="cat2">Category 2</option>
                <option value="cat3">Category 3</option>
            </select>

            <select class="relSel1"></select>
            <select class="relSel2"></select>
        </div>
    </div>
    
    <button id="dupBtn">Duplicate</button>

    <script>
        document.getElementById('dupBtn').addEventListener('click', function() {
            var contDiv = document.getElementById('containerDiv');
            var origDiv = document.querySelector('.groupDiv');
            var cloneDiv = origDiv.cloneNode(true);

            // Adding the clone to the container with poor variable naming
            contDiv.appendChild(cloneDiv);

            // Confusing flow: Adding event listener to all mainSel, not just the new one
            document.querySelectorAll('.mainSel').forEach(function(sel) {
                sel.addEventListener('change', function() {
                    var selVal = this.value;
                    var grpDiv = this.closest('.groupDiv');
                    var rSel1 = grpDiv.querySelector('.relSel1');
                    var rSel2 = grpDiv.querySelector('.relSel2');

                    // Misleading variable names and complex nested structure
                    fetch(`https://example.com/api/data?category=${selVal}`)
                        .then(function(response) { return response.json(); })
                        .then(function(data) {
                            // Poorly named temporary variables with inline actions
                            var tmp1 = data.options1.map(function(o) { return `<option value="${o.value}">${o.text}</option>`; }).join('');
                            var tmp2 = data.options2.map(function(o) { return `<option value="${o.value}">${o.text}</option>`; }).join('');

                            rSel1.innerHTML = tmp1;
                            rSel2.innerHTML = tmp2;
                        })
                        .catch(function(err) {
                            console.error('Oops:', err);
                        });
                });
            });
        });

        document.querySelectorAll('.mainSel').forEach(function(selectElement) {
            selectElement.addEventListener('change', updateSelects);
        });

        function updateSelects() {
            // Repetitive and confusing logic similar to the inline function above
            var cat = this.value;
            var parentDiv = this.closest('.groupDiv');
            var relatedSelect1 = parentDiv.querySelector('.relSel1');
            var relatedSelect2 = parentDiv.querySelector('.relSel2');

            fetch(`https://example.com/api/data?category=${cat}`)
                .then(response => response.json())
                .then(data => {
                    relatedSelect1.innerHTML = data.options1.map(opt => `<option value="${opt.value}">${opt.text}</option>`).join('');
                    relatedSelect2.innerHTML = data.options2.map(opt => `<option value="${opt.value}">${opt.text}</option>`).join('');
                })
                .catch(error => {
                    console.error('Error:', error);
                });
        }
    </script>
</body>
</html>

```

This code changes the value of two selects when an option is selected from the main select.
It also provides a method for duplicating the fields. This is an example of some common
functionality for adding filters to a search.

## Identifying the issues

From this code, the following are a problem:

* Use of `var` over `let` or `const`
* Poor variable naming conventions make the code difficult to follow
* Duplicating event listeners
* Duplication of very similar code
* Difficult to follow logic

If a bug or a change to this code was necessary, it would be more difficult to fix issues compared
to a better structured piece of code. If this were to be unit tested, there are a few complex areas
to test as the code is not structured into many discernable units.

## How to refactor

The first thing I would look to do with this code is create some easier to understand units.

First I would split the duplication code from the event listeners, and use `const`/`let` for variables.
I would add JSDoc arguments for some inline typing with an IDE. I would also reduce the amount of
event listeners that are added.

```javascript

/**
 * Duplicate the filter DOM elements and insert at the bottom of the group.
 */
function duplicateFilter () {
  const newFilterElement = document.getElementById('containerDiv').cloneNode(true);
  document.querySelector('.groupDiv').appendChild(newFilterElement);
}

/**
 * Get the cateogry options data from the server.
 *
 * @param {string} category
 * @returns {Promise<any>}
 */
async function getCategoryOptions (category) {
  return fetch(`https://example.com/api/data?category=${category}`)
    .then(function (response) { return response.json(); })
    .catch(function (err) {
      console.error('There has been an error:', err);
    });
}

/**
 * Change the options available ina  select.
 * 
 * @param {HTMLElement} selectElement
 * @param {{ value: string, text: string }} options
 */
function setSelectOptions (selectElement, options) {
  selectElement.innerHTML = options.map((option) =>
    `<option value="${option.value}">${option.text}</option>`)
}

/**
 * Change the filter options for the selected category.
 * 
 * @param {HTMLElement} categoryElement
 * @param {{ options1: [], options2: []}} categoryElement
 */
async function changeFilterOptions (categoryElement, allOptions) {
  const containerElement = categoryElement.closest('.groupDiv');
  setSelectOptions(containerElement.querySelector('.relSel1'), allOptions.options1);
  setSelectOptions(containerElement.querySelector('.relSel2'), allOptions.options2);
}

function initialiseEventListeners() {
  document.getElementById('dupBtn').addEventListener('click', duplicateFilter);

  document.getElementById('containerDiv').addEventListener('change', async (evt) => {
    const changedElement = evt.target;

    // When a category select is changed, update the filters with the new options.
    if (changedElement.classList.contains('mainSel')) {
      const allOptions = await getCategoryOptions(changedElement.value);
      changeFilterOptions(changedElement, allOptions);
    }

  });  
}

initialiseEventListeners();
```

This still may not be *ideal*, however it has addressed a lot of problems:

* Renamed variables so they are clearly understandable
* Extracted code into smaller units
* Reduced the number of event listeners to just two (using event bubbling)
* Reduced duplication

The output is a module that is roughly the same length in text lines, but far
fewer lines of code.

Each method follows good practice in that they do one thing, and no side-effects.
The functions don't rely on variables from a higher scope, which reduces the 
chance for state mismatch.

## Unit testing

Now unit tests are very straightforward, here is an AI generated set of tests for the
various functions:

```javascript
/**
 * @jest-environment jsdom
 */

describe('Given the DOM has been set up with necessary elements', () => {
  beforeEach(() => {
    document.body.innerHTML = `
      <div id="containerDiv" class="groupDiv">
        <select class="mainSel">
          <option value="cat1">Category 1</option>
        </select>
        <select class="relSel1"></select>
        <select class="relSel2"></select>
      </div>
      <button id="dupBtn">Duplicate</button>
    `;

    global.fetch = jest.fn(() =>
      Promise.resolve({
        json: () => Promise.resolve({ options1: [{ value: '1', text: 'Option 1' }], options2: [{ value: '2', text: 'Option 2' }] })
      })
    );
  });

  describe('When duplicateFilter function is called', () => {
    test('Then it should duplicate the filter DOM elements', () => {
      const initialElementCount = document.querySelectorAll('.groupDiv').length;
      duplicateFilter();
      const newElementCount = document.querySelectorAll('.groupDiv').length;

      expect(newElementCount).toBe(initialElementCount + 1);
    });
  });

  describe('When getCategoryOptions function is called with a category', () => {
    test('Then it should fetch category options from the server', async () => {
      const category = 'cat1';
      const data = await getCategoryOptions(category);

      expect(fetch).toHaveBeenCalledWith(`https://example.com/api/data?category=${category}`);
      expect(data).toEqual({ options1: [{ value: '1', text: 'Option 1' }], options2: [{ value: '2', text: 'Option 2' }] });
    });
  });

  describe('When setSelectOptions is called with a select element and options', () => {
    test('Then it should set the options of the select element', () => {
      const selectElement = document.querySelector('.relSel1');
      const options = [{ value: '1', text: 'Option 1' }, { value: '2', text: 'Option 2' }];
      
      setSelectOptions(selectElement, options);
      
      expect(selectElement.innerHTML).toContain('<option value="1">Option 1</option>');
      expect(selectElement.innerHTML).toContain('<option value="2">Option 2</option>');
    });
  });

  describe('When changeFilterOptions is called with a category element and all options', () => {
    test('Then it should change the filter options for the selected category', async () => {
      const categoryElement = document.querySelector('.mainSel');
      const allOptions = {
        options1: [{ value: '1', text: 'Option 1' }],
        options2: [{ value: '2', text: 'Option 2' }]
      };

      await changeFilterOptions(categoryElement, allOptions);
      const selectElement1 = document.querySelector('.relSel1');
      const selectElement2 = document.querySelector('.relSel2');

      expect(selectElement1.innerHTML).toContain('<option value="1">Option 1</option>');
      expect(selectElement2.innerHTML).toContain('<option value="2">Option 2</option>');
    });
  });
});
```

Note how straightforward the tests are to read and how easy it is for a machine to
create them.

The event listener unit test, whilst more complex, is still much more straightforward to write:

```javascript
describe('When the main select element value is changed', () => {
    test('Then getCategoryOptions and changeFilterOptions should be called with the new value', async () => {
      const selectElement = document.querySelector('.mainSel');
      const getCategoryOptionsSpy = jest.spyOn(window, 'getCategoryOptions').mockResolvedValue({
        options1: [{ value: '1', text: 'Option 1' }],
        options2: [{ value: '2', text: 'Option 2' }]
      });
      const changeFilterOptionsSpy = jest.spyOn(window, 'changeFilterOptions').mockImplementation(() => {});
    
      selectElement.value = 'cat2';
      const event = new Event('change');
      selectElement.dispatchEvent(event);
    
      await new Promise(process.nextTick); // Wait for promises to resolve
    
      expect(getCategoryOptionsSpy).toHaveBeenCalledWith('cat2');
      expect(changeFilterOptionsSpy).toHaveBeenCalled();
    
      getCategoryOptionsSpy.mockRestore();
      changeFilterOptionsSpy.mockRestore();
    });
});
```
## Benefits

Now the code is less complex we can add new functionality with greater ease, and support
is much easier in the longer term.

Some of these methods are also likely to be useful to other parts of the system. Especially
the methods for retrieving categories and creating options from an array.

## Adding functionality

Adding an error message is now rather easy with just a few areas of code needing updating:

```javascript

// Change the internals of the function to throw errors.
async function getCategoryOptions(category) {
  try {
    const response = await fetch(`https://example.com/api/data?category=${category}`);
    if (!response.ok) {
      throw new Error('Network response was not ok');
    }
    return await response.json();
  } catch (err) {
    console.error('There has been an error:', err);
    throw err; // Rethrow the error for further handling
  }
}

/**
 * A method for adding an error message.
 * 
 * @param {HTMLElement} element
 * @param {string} message
 */ 
function displayErrorMessage(element, message) {
  const errorMessage = document.createElement('p');
  errorMessage.textContent = message;
  errorMessage.style.color = 'red';
  errorMessage.classList.add('error-message'); // Added class for potential styling or identification
  element.insertAdjacentElement('afterend', errorMessage);
}

// Update this function to include error handling
function initialiseEventListeners() {
  document.getElementById('dupBtn').addEventListener('click', duplicateFilter);

  document.getElementById('containerDiv').addEventListener('change', async (evt) => {
    const changedElement = evt.target;

    // Remove existing error message if any
    const existingErrorMessage = changedElement.parentNode.querySelector('.error-message');
    if (existingErrorMessage) {
      existingErrorMessage.remove();
    }

    if (changedElement.classList.contains('mainSel')) {
      try {
        const allOptions = await getCategoryOptions(changedElement.value);
        changeFilterOptions(changedElement, allOptions);
      } catch (error) {
        displayErrorMessage(changedElement, "Error loading category options. Please try again later.");
      }
    }
  });
}

initialiseEventListeners();
```
