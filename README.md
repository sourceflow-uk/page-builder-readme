<div align="center">
  <a href="https://github.com/sourceflow-uk/page-builder-readme/">
    <img src="https://github.com/user-attachments/assets/626e049f-6030-4726-909a-a4305346d2c6" alt="SourceFlow Page Builder Logo" width="300" height="300">
  </a>

  <h3 align="center">SourceFlow Page Builder</h3>

  <p align="center">
    An internal framework for Next.js sites to be dynamically constructible via the CMS
    <br />
    <a href="https://github.com/sourceflow-uk/page-builder-readme/"><strong>Explore the docs »</strong></a>
  </p>
</div>

# SourceFlow Page Builder README

This guide explains how to integrate your project with the SourceFlow Page Builder, including setup, usage, and best practices.

## Table of Contents

- [SourceFlow Page Builder README](#sourceflow-page-builder-readme)
  - [Table of Contents](#table-of-contents)
  - [Concept](#concept)
  - [1. Setup](#1-setup)
    - [1. Install the CLI](#1-install-the-cli)
    - [2. ESLint Configuration](#2-eslint-configuration)
    - [3. Add Build Scripts](#3-add-build-scripts)
    - [4. CLI Flags](#4-cli-flags)
  - [2. Writing Definitions Files](#2-writing-definitions-files)
    - [Required Fields](#required-fields)
      - [Example Definition](#example-definition)
    - [Prop Schema Fields](#prop-schema-fields)
      - [Prop Default Values Example](#prop-default-values-example)
      - [Prop Supported Types](#prop-supported-types)
      - [Example for Nested Templates and Objects](#example-for-nested-templates-and-objects)
      - [Full example of a `definitions.sourceflow.mjs` file](#full-example-of-a-definitionssourceflowmjs-file)
    - [Using `sfgenerate` to generate scaffold files](#using-sfgenerate-to-generate-scaffold-files)
  - [3. Creating the Components Page](#3-creating-the-components-page)
    - [Key Requirements](#key-requirements)
      - [Example Editable Element Click Handler](#example-editable-element-click-handler)
  - [4. Next.js Dynamic Routing](#4-nextjs-dynamic-routing)
  - [Final Notes](#final-notes)

## Concept

The SourceFlow Page Builder lets you define reusable components and manage pages visually. Here’s how it works:

1. **Component Definitions**:  
   For each component you want to use in the Page Builder, create a `definitions.sourceflow.mjs` file.

2. **Build Step**:  
   Use the `sfprepare` CLI (installed via NPM) to combine all definitions into a single `sourceflow-components.json` file.

3. **Accessing Definitions**:  
   The combined JSON file is available at `<<domain>>/sourceflow-components.json`.

4. **Components Page**:  
   Create a `components.jsx` page that outputs a static `components.html` file (client-side only) at `<<domain>>/components`.

5. **Dynamic Routing**:  
   Update your `[url_slug].js` catch-all route to extract data from Page Builder and render pages accordingly.

6. **CMS Integration**:  
   SourceFlow CMS will embed your `components.html` page in an iframe and communicate via `postMessage`.

7. **Data Structure**:  
   The CMS sends an array of objects, each with an `id`, `props`, and `component` name, for rendering:

   ```js
   [
     {
       id: '0d3b1b32-bae9-4dc4-948b-9bb28542e62d',
       props: {
         title: 'Enter the main title here...',
         content: '<p>Enter formatted content here...</p>',
         className: '',
       },
       component: 'Content',
     },
   ];
   ```

8. **Rendering**:  
   The `components.html` page renders components based on the received data.

9. **Workflow Summary**:
   - Write definitions files
   - Combine them with `sfprepare`
   - CMS picks up the definitions
   - CMS loads your components page
   - CMS sends content data
   - Components page renders a preview
   - User saves
   - Site renders saved data server-side

## 1. Setup

### 1. Install the CLI

Install the CLI tool in your project:

```sh
npm i @sourceflow-uk/page-builder-cli
# or
yarn add @sourceflow-uk/page-builder-cli
# or
pnpm add @sourceflow-uk/page-builder-cli
```

All projects that use SourceFlow's Page Builder must install this package.

### 2. ESLint Configuration

Add this to your ESLint config (e.g., `.eslintrc`, `.eslintrc.json`) to avoid warnings for anonymous default exports in definitions files:

```json
"overrides": [
  {
    "files": ["**/definitions.sourceflow.mjs"],
    "rules": {
      "import/no-anonymous-default-export": "off"
    }
  }
]
```

### 3. Add Build Scripts

Update your `package.json`:

```json
"scripts": {
  "postbuild": "sfprepare -d /components -o /out"
}
```

Or, if you want a separate script:

```json
"scripts": {
  "sfprepare": "sfprepare -d /components -o /out",
  "postbuild": "npm run sfprepare"
}
```

**Notes:**

- Do **not** use `prepare` as the script name (it's a reserved NPM lifecycle script).
- If you already have a `postbuild` script (e.g., for `next-sitemap`), add `sfprepare` before other commands:

  ```json
  "postbuild": "sfprepare -d /components -o /out && next-sitemap"
  ```

### 4. CLI Flags

| Flag              | Description                                       | Default                      | Example                             |
| ----------------- | ------------------------------------------------- | ---------------------------- | ----------------------------------- |
| `-V, --version`   | Show version number                               | -                            | `sfprepare -V`                      |
| `-d, --directory` | Components folder to scan for definitions         | `/src/components`            | `sfprepare -d /components`          |
| `-p, --preview`   | Path to the components preview HTML               | `/out/components/index.html` | `sfprepare -p /out/components.html` |
| `-o, --output`    | Output directory for `sourceflow-components.json` | `/out`                       | `sfprepare -o /out`                 |
| `-h, --help`      | Show help                                         | -                            | `sfprepare -h`                      |

**Example:**

```sh
sfprepare -d /components -o /dist
```

If your Next.js project outputs to `/out`, you can omit the `-o` flag.

---

## 2. Writing Definitions Files

- Only components with a `definitions.sourceflow.mjs` file will appear in the Page Builder.
- The file **must** be named exactly `definitions.sourceflow.mjs`.

### Required Fields

| Field                | Description                  | Required                                                                                    | Example                          |
| -------------------- | ---------------------------- | ------------------------------------------------------------------------------------------- | -------------------------------- |
| `component`          | Name matching your component | Yes                                                                                         | `BlogSection`                    |
| `label`              | User-friendly name           | Yes                                                                                         | `Blog Section`                   |
| `component_category` | Category in snake_case       | Yes                                                                                         | `default`                        |
| `propSchema`         | Array of prop definitions    | Yes (if your component has at least 1 prop, if there are no props, you can omit this field) | [See below](#example-definition) |

#### Example Definition

```js
export default {
  component: 'BlogSection',
  label: 'Blog Section',
  component_category: 'default',
  propSchema: [
    {
      name: 'title',
      label: 'Title',
      type: 'string',
      defaultValue: 'Enter the main title here...',
    },
    {
      name: 'className',
      label: 'Class Name',
      type: 'string',
      defaultValue: '',
    },
    {
      name: 'description',
      label: 'Description',
      type: 'formatted_text',
    },
    {
      name: 'icon',
      label: 'Icon',
      type: 'file',
    },
    // ...more props
  ],
};
```

### Prop Schema Fields

Each prop in `propSchema` can have the following fields:

| Field                            | Required?                 | Description                                                         | Example / Values                                                 |
| -------------------------------- | ------------------------- | ------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `name`                           | Yes                       | Prop name (must match your component)                               | `"title"`                                                        |
| `label`                          | Yes                       | User-friendly label                                                 | `"Section with Image"`                                           |
| `type`                           | Yes                       | Data type                                                           | See [Supported Types](#prop-supported-types) below               |
| `template_schema`                | Yes (if `type: template`) | Defines fields for nested templates                                 | See example below                                                |
| `object_schema`                  | Yes (if `type: object`)   | Defines fields for nested objects                                   | See example below                                                |
| `defaultValue`                   | No                        | Default value for the field                                         | See [Default Values Example](#prop-default-values-example) below |
| `category_internal_build_name`   | No                        | Internal build name for the category (snake_case)                   | `"testimonials"`                                                 |
| `category_id`                    | No                        | ID of the category that you want to link this prop to (UUID string) | `"cb62ab14-dfac-43f6-89ab-a283445584a4"`                         |
| `allowMultipleOptions`           | No                        | Allow multiple options (for `array` or `category_id`)               | `true` / `false`                                                 |
| `options`                        | No                        | Options for select fields                                           | `[ { label: "A", value: "A" } ]`                                 |
| `isRequired` (Not supported yet) | No                        | Mark as required                                                    | `true` / `false`                                                 |

#### Prop Default Values Example

- **String**: `"MyDefaultStringValue"` In a string format.
- **Number**: `123` In a number format.
- **Boolean**: `true` / `false` In a boolean format.
- **Array**: `["A", "B", "C"]` or `[{any: "thing"}]` In an array format.
- **File**: `{id: "<<ID_OF_IMAGE_FROM_CMS>>"}` or `{url: "https://example.com/image.jpg"}` In an object either with `id` or `url`.
- **Template**: `[{any: "thing"}]`. In an array format.
- **Formatted Text**: `"<p>My default formatted text</p>"` In a string format.
- **Forms**: _Not supported_.
- **Object**: `{any: "thing"}` In an object format.
- **Modules**: _Not supported_.
- **Repeater**: `[{any: "thing"}]` In an array format.

#### Prop Supported Types

- `string`
- `number`
- `boolean`
- `array`
- `file`
- `template`
- `formatted_text`
- `forms`
- `object`
- `modules`
- `repeater`

**Notes:**

- Use `type: 'repeater'` for simple repeatable fields (e.g., multiple strings).
- Use `type: 'template'` and `template_schema` for more complex/nested repeatable set of fields.
- Use `type: 'object'` and `object_schema` for nested objects (single instance).

#### Example for Nested Templates and Objects

```js
export default {
  component: 'CTA',
  label: 'Call to Action',
  component_category: 'default',
  propSchema: [
    {
      name: 'myobject',
      label: 'My Object',
      type: 'object',
      object_schema: [
        { name: 'field1', label: 'Field 1', type: 'string' },
        {
          name: 'field2',
          label: 'Field 2',
          type: 'template', // We can even put templates inside objects
          template_schema: [
            { name: 'something', label: 'Something', type: 'string' },
            { name: 'somethingElse', label: 'Something Else', type: 'number' },
          ],
        },
      ],
    },
    {
      name: 'mytemplate',
      label: 'My Template',
      type: 'template',
      template_schema: [
        // ...other fields
        {
          label: 'Field1 - String linked with Category Example',
          name: 'field1',
          category_id: 'cb62ab14-dfac-43f6-89ab-a283445584a4',
          allowMultipleOptions: true,
          type: 'string',
          defaultValue: 'Hello world!',
        },
        {
          name: 'Field8 - Object Example',
          label: 'Field8',
          type: 'object', // We can even put objects inside templates
          object_schema: [
            {
              name: 'Field8.1',
              label: 'Field8.1',
              type: 'string',
              defaultValue: 'Hello world!',
            },
            // ... more nested fields
          ],
        },
      ],
      isRequired: false,
    },
  ],
};
```

#### Full example of a `definitions.sourceflow.mjs` file

```js
export default {
  component: 'ExampleComponent',
  label: 'Example Component',
  component_category: 'default',
  propSchema: [
    { name: 'title', label: 'title', type: 'string', defaultValue: 'Hello world!' },
    {
      name: 'description',
      label: 'description',
      type: 'formatted_text',
    },
    { name: 'buttonText', bs_col_width: 4, type: 'string', defaultValue: 'Click me!', isRequired: true },
    {
      name: 'buttonClass',
      bs_col_width: 4,
      type: 'string',
      defaultValue: '',
      options: [
        {
          label: 'Primary',
          value: 'btn-primary',
        },
        {
          label: 'Secondary',
          value: 'btn-secondary',
        },
        {
          label: 'Danger',
          value: 'btn-danger',
        },
      ],
      allowMultipleOptions: false,
      isRequired: true,
    },
    {
      name: 'sponsors',
      type: 'repeater',
    },
    {
      name: 'icon',
      label: 'icon',
      type: 'file',
    },
    { name: 'myRandomNameForModuleID', type: 'modules' },
    {
      name: 'template',
      type: 'template',
      template_item_label: 'Module Items',
      template_schema: [
        {
          label: 'Field1',
          name: 'field1',
          category_id: 'cb62ab14-dfac-43f6-89ab-a283445584a4',
          allowMultipleOptions: true, // Allow multiple category values to be selected
        },
        {
          label: 'Field2Module',
          name: 'field2Module',
          type: 'modules',
        },
      ],
    },
    {
      name: 'massive_object_one',
      label: 'massive_object_one',
      type: 'object',
      object_schema: [
        {
          label: 'Field 1',
          name: 'image1',
          type: 'file',
          defaultValue: {
            id: '32248875-332a-4111-8a1c-f5e1a3752936',
          },
        },
      ],
    },
    {
      name: 'massive_object_two',
      label: 'massive_object_two',
      type: 'object',

      object_schema: [
        {
          label: 'Field 1',
          name: 'image1',
          type: 'file',
          defaultValue: {
            id: '32248875-332a-4111-8a1c-f5e1a3752936',
          },
        },
      ],
    },
    {
      name: 'massive_object_three',
      label: 'massive_object_three',
      type: 'object',
      object_schema: [
        {
          label: 'Field 1',
          name: 'image1',
          type: 'file',
          defaultValue: { url: 'https://placehold.co/300' },
        },
      ],
    },
    {
      name: 'blogs',
      label: 'blogs',
      type: 'template',
      template_item_label: 'Blog',
      template_schema: [
        {
          label: 'Field0 - Repeater',
          name: 'Field0-Repeater',
          type: 'repeater',
        },
        {
          label: 'Field1',
          name: 'field1',
          category_id: 'cb62ab14-dfac-43f6-89ab-a283445584a4',
          allowMultipleOptions: true,
          type: 'string',
          defaultValue: 'Hello world!',
        },
        {
          label: 'Field2',
          name: 'field2',
          type: 'number',
          defaultValue: 0,
        },
        {
          label: 'Field3',
          name: 'field3',
          type: 'boolean',
          defaultValue: false,
        },
        {
          label: 'Field4',
          name: 'field4',
          type: 'array',
          options: [],
          defaultValue: [],
        },
        {
          label: 'Field5',
          name: 'field5',
          type: 'file',
        },
        {
          label: 'Field6',
          name: 'field6',
          type: 'formatted_text',
          defaultValue: '',
        },
        {
          label: 'Field 7',
          name: 'field7',
          type: 'object',
          object_schema: [{ label: 'Field 7.1', name: 'field7.1', type: 'string', defaultValue: 'Hello world!' }],
        },
        {
          label: 'Field 8',
          name: 'field8',
          type: 'template',
          template_schema: [{ label: 'Field 8. name', name: 'name', type: 'string', defaultValue: 'Hello world!' }],
        },
      ],
    },
  ],
};
```

### Using `sfgenerate` to generate scaffold files

Generate a boilerplate definitions file:

```sh
sfgenerate -d /components -n MyComponent
```

```sh
sfgenerate --directory /path/to/output field1:string field2:number field3:array field4:boolean field5:file field6:template field7:formatted_text field8:forms field9:object
```

**Flags:**

| Flag                      | Description                     | Default           | Example                         |
| ------------------------- | ------------------------------- | ----------------- | ------------------------------- |
| `-V, --version`           | Show version                    | -                 | `sfgenerate -V`                 |
| `-d, --directory`         | Output directory                | `/src/components` | `sfgenerate -d /components`     |
| `-n, --name`              | Component name                  | -                 | `sfgenerate -n MyComponent`     |
| `-do, --definitions-only` | Only output definitions file    | false             | `sfgenerate -do -n MyComponent` |
| `-c, --with-docs`         | Include documentation in output | false             | `sfgenerate -c -n MyComponent`  |
| `-h, --help`              | Show help                       | -                 | `sfgenerate -h`                 |

**Reminder:**
Review and clean up the generated boilerplate as needed.

## 3. Creating the Components Page

- Output to `/out/components/index.html` or `/out/components.html`.
- Place the page in your `pages` or `app` directory (e.g., `pages/components.jsx`).

### Key Requirements

1. **Content State**

   ```js
   const [content, setContent] = useState([]);
   ```

2. **Receiving Messages**

   Use `useEffect` to listen for messages from the parent CMS:

   ```js
   useEffect(() => {
     window.addEventListener('message', (event) => {
       if (!event.data) return;
       const parsed = JSON.parse(event.data.message);

       switch (event.data.type) {
         case 'SET_CONTENT':
           setContent(parsed);
           break;
         // Handle FOCUS_CONTENT and CLEAR_FOCUS_CONTENT as needed
         default:
           console.log('Unknown event type.');
       }
     });
   }, []);
   ```

   **Event Types:**

   - `SET_CONTENT`: Sets the content array for rendering.
   - `FOCUS_CONTENT`: Focuses/highlights a specific element by ID.
   - `CLEAR_FOCUS_CONTENT`: Removes highlight from a specific element.

3. **Sending Messages (Optional but recommended for user experience)**

   You can send messages back to the CMS for actions like focusing on a specific prop field. This is optional but improves user experience.

   | `type`                             | Description                                                                                              | Payload                             | Example                                              |
   | ---------------------------------- | -------------------------------------------------------------------------------------------------------- | ----------------------------------- | ---------------------------------------------------- |
   | `type: 'EDITABLE_ELEMENT_CLICKED'` | Sends the ID and prop name of the clicked element, this will trigger the CMS to focus on that prop field | `{ id: contentId, propName: prop }` | [See below](#example-editable-element-click-handler) |

   #### Example Editable Element Click Handler

   ```js
   document.body.addEventListener('click', (event) => {
     const prop = event.target.getAttribute('data-sourceflow-prop');
     if (prop) {
       const element = event.target.closest('[data-sourceflow-content-id]');
       if (element) {
         const contentId = element.getAttribute('data-sourceflow-content-id');
         window.parent.postMessage(
           {
             type: 'EDITABLE_ELEMENT_CLICKED',
             message: { id: contentId, propName: prop },
           },
           '*'
         );
       }
     }
   });
   ```

   ```js
   const style = document.createElement('style');
   style.innerHTML = `
     [data-sourceflow-prop]:hover {
       border: 2px dashed gray;
       cursor: pointer;
     }
   `;
   document.head.appendChild(style);
   ```

4. **Rendering Components**

   Example:

   ```js
   return (
     <div>
       <Content items={content} />
     </div>
   );
   ```

   Where `<Content />` maps over the items and renders the correct component by name:

   ```js
   export default function Content({ items, global }) {
     const allowedComponents = {
       Accordion,
       ArticleFeed,
       BlockQuote,
       Divider,
       Form,
       TeamBio,
       Title,
     };

     return (
       <section>
         {items.filter(Boolean).map(({ id, component, props }) => (
           <div key={id}>
             <a id={id} />
             {(() => {
               const Component = allowedComponents[component];
               return Component ? <Component key={id} global={global} {...props} /> : null;
             })()}
           </div>
         ))}
       </section>
     );
   }
   ```

---

## 4. Next.js Dynamic Routing

Update your `[url_slug].js` file to read from `dynamic_pages.json`:

```js
import Content from '../components/Content';
import dynamic_pages from '../.sourceflow/dynamic_pages.json';

export default function Page({ content }) {
  return <Content items={content} />;
}

export async function getStaticProps({ params }) {
  const { url_slug } = params;
  const slugPath = url_slug.join('/');
  const page = dynamic_pages.find((page) => page.url_slug === slugPath);

  return {
    props: {
      meta: { title: page.title },
      content: page.content,
    },
  };
}

export async function getStaticPaths() {
  const paths = dynamic_pages.map((page) => ({
    params: { url_slug: page.url_slug.split('/') },
  }));

  return {
    paths,
    fallback: false,
  };
}
```

**Tips:**

- Ensure dynamic pages overwrite existing routes as needed.

## Final Notes

- Only components with a `definitions.sourceflow.mjs` file are available in Page Builder.
- Always check and clean up generated files.
- Ensure your build scripts and output paths are correct.
