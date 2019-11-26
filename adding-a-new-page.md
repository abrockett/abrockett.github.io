Creating a New Page
===================
We will use the User Stories page as an example throughout this document. The source code for this page can be found in `~/projects/churro/src/pages/user-stories` 

# Prerequisites
- Churro environment is set up. This document will assume a working development environment for churro is up and running. [See here for instructions](https://github.com/RallySoftware/churro#quick-start).
- Feature Toggle: typically new pages are implemented behind a feature toggle so that we can control the release of functionality independent of our deploy cadences.
    -  For more information regarding feature toggles and how to create a new one, look [here](https://github.com/RallySoftware/alm#feature-toggles)

# Let's Do This Thing
1. Pages should be created in the `src/pages` directory by creating a new directory. In our example, our directory will be called: `user-stories`
    - Some Notes About Directory Structure
        - Minimal example directory structure:
        ```
          pages
            - user-stories
              - components
              - containers
              index.js
              UserStoriesPage.js
        ```
        - `components` should contain React component modules
        - `containers` should contain an imported component connected to the Redux store via react-redux's `connect` or one of our internal higher-order components (such as `withSchema`)
            - Note: Container modules names should end in `Container.js` (e.g. `UserStoriesPageContainer.js`)
            - Note: Container modules should not contain React component definitions, React components should be defined in a separate file.
1. Minimum Requirements
    - `index.js`
        - Must fufill the following interface:
        ```javascript
        /* index.js */
        export default {
          /**
           * Given the current slug and application state from Redux, this function
           * should return a boolean indicating whether or not this page should
           * be the one rendered for the current slug.
           *
           * @param {String} slug The current route that the user has navigated to
           * @param {Object} reduxState The current state of the application
           *
           * @returns {boolean} Whether this page should be displayed for the supplied `slug`
           */
          matches(slug, reduxState) { /* ... */ },
      
          /**
           * Given the current navigationState and the Redux store, this function should return
           * a Promise that resolves to a React Component.
           *
           * @param {NavigationState} navigationState
           * @param {ReduxStore} reduxStore The Redux store, generally used here to either
           * `store.dispatch` actions or read values from `store.getState()`
           *
           * @returns {Promise} Returns a Promise that resolves the React Component to be rendered for this page.
           */
          initPage(navigationState, reduxStore) { /* ... */ },
      
         /**
          * @Legacy
          * This legacy method should generally return an empty array for newly created pages. 
          * If you're not sure if you should return something here, you probably do not need to.
          *
          * For legacy pages, this may need to return an array containing one or
          * more strings indicating legacy frameworks the page requires.
          * For example, a page that requires ExtJS code from our Pinata repository
          * would need to return an array such as: `['js!ext4rally']`
          */
          getDependencies(reduxState, navigationState) { /* ... */ } 
        };
        ``` 
        - Now let us take a look at the User Stories `index.js` for a full example: 
            - Note how the `matches` function below checks a feature toggle in addition to the slug to determine if this page should be rendered.
        ```javascript
        /* src/pages/user-stories/index.js */
       
        import dynamicallyLoad from 'utils/dynamicallyLoad';
        import { isToggleEnabled } from 'reducers/UserContextReducer';
        // When webpack is upgraded to >= v4.x.x, we'll replace this deprecated
        // loader syntax and promise-loader package with dynamic imports, which 
        // are the modern equivalent for lazy-loading an ecmascript module.
        //
        // eslint-disable-next-line import/no-webpack-loader-syntax
        const UserStoriesPage = require('promise-loader?global,userstories!./UserStoriesPage');
        
        export default {
          matches(slug, state) {
            return (
              slug === '/userstories' &&
              isToggleEnabled(state, 'F20161_REACT_USER_STORIES_PAGE')
            );
          },
        
          initPage() {
            return dynamicallyLoad(UserStoriesPage);
          },
        
          getDependencies() {
            return [];
          },
        };
        ```
    - The Page (e.g. `UserStoriesPage.js`)
        - Must fufill the following interface:
          ```javascript    
          export default {
            /**
             * The only required method for this module
             * @returns {React Component} Should return a React component
             */
            getReactComponent() { /* ... */ },
          
            /**
             * Optional.
             * Can be used to set a document title for the page. If a dynamic document
             * title is required, see the DocumentTitle component
             * @returns {String} Should return an externalized string to use as the document title. 
             * See [here](http://churro-docs.tools.f4tech.com/guides/i18n.html) for more info on 
             * how we do internationalization.
             */
            getTitle() { /* ... */ }
          };
          ```
        - Looking at User Stories as an example:
            ```javascript
            import UserStoriesPageContainer from './containers/UserStoriesPageContainer';
            import { formatMessageAsString } from 'i18n/StringFormatting';
            
            export default {
              getReactComponent() {
                return UserStoriesPageContainer;
              },
            
              getTitle() {
                return formatMessageAsString({ messageKey: 'UserStoriesPage/title' });
              },
            };
            ```
    - The React Component
        - Implied by `getReactComponent` above, is the need for an actual React component to render some content for the page.
        -  Here is an annotated example of the User Stories page's `Page.js`:
        ```javascript
        import PropTypes from 'prop-types';
        import React from 'react';
        import * as ImmutablePropTypes from 'react-immutable-proptypes';
        import { LoadingMask, PageHeader } from 'components/common/index';
        import { formatMessageAsString } from 'i18n/StringFormatting';
        import LayoutPage from 'components/layout/Page';
        import Schema from 'utils/alm/Schema';
        import { genGetClassName } from 'utils/Namespacing';
        import SavedViewsSelect from 'components/saved-views/SavedViewsSelect';
        import FeedbackButton from 'components/feedback-button/FeedbackButton';
        import UserStoriesDataTable from './UserStoriesDataTable';
        
        // We primarily use SASS for our styles, with some exceptions
        // where React's inline styles are used. 
        import './Page.scss';
        
        
        const displayName = 'UserStoriesPage';
        // CSS class names should be generated via our `getClassName` utility
        // which is based on SUIT CSS namespacing.
        const getClassName = genGetClassName(displayName);
        const userStoriesFeedbackId = 'userstories';
        
        export default class Page extends React.Component {
          // It is important that all components have a displayName property
          // as this property is utilized for our client metrics infrastructure
          static displayName = displayName;
        
          static propTypes = {
            schema: PropTypes.instanceOf(Schema),
            scope: ImmutablePropTypes.map.isRequired,
          };
        
          // Internationalized strings are best defined once outside of the component's
          // render method, unless they take arguments for value substition (such as a
          // count for pluralization).
          pageTitle = formatMessageAsString({ messageKey: 'UserStoriesPage/title' });
        
          i18n = {
            pageTitle: this.pageTitle,
            feedbackSubject: formatMessageAsString({
              messageKey: 'UserStoriesPage/feedback',
            }),
            feedbackTitle: formatMessageAsString({
              messageKey: 'Shared/feedback-on-new-page',
              title: this.pageTitle,
            }),
          };
        
          render() {
            const feedbackButtonProps = {
              feedbackId: userStoriesFeedbackId,
              subject: this.i18n.feedbackSubject,
              title: this.i18n.feedbackTitle,
            };
        
            return (
              {/* It is recommended to use LayoutPage to ensure a uniform look 
                * and (you guess it) layout for all of our React pages. */}
              <LayoutPage className={getClassName()}>
                {/* The PageHeader component is the recommended way of getting 
                  * consistent layout of UI elements in the Header of a Page's 
                  * content. This may include a feedback button as in this example
                  * as well as potentially a HelpButton or SavedViews select */}
                <PageHeader title={this.i18n.pageTitle}>
                  <PageHeader.HeaderGroup position="left">
                    <SavedViewsSelect />
                  </PageHeader.HeaderGroup>
                  <PageHeader.HeaderGroup position="right">
                    <FeedbackButton {...feedbackButtonProps} />
                  </PageHeader.HeaderGroup>
                </PageHeader>
                {/* See below for comments on main content rendering logic */}
                {this.renderView()}
              </LayoutPage>
            );
          }
        
          renderView() {
            const {
              changeViewModel,
              viewModel,
              schema,
              scope,
              isSavedView,
            } = this.props;
        
            // It is generally advisable to defer attempting to render any of the
            // page's main content before the schema is available. Most of our
            // UI is generated based on the Types and Attributes defined in the
            // schema, so without it we generally can't show much of anything 
            // useful anyway, so we typically render a LoadingMask until it is
            // available. 
            // The same can be said of the `viewModel` for pages utilizing
            // saved views
            if (!schema || !viewModel) {
              return <LoadingMask />;
            }
        
            const userStoriesDataTableProps = {
              changeDataTableModel: changeViewModel,
              dataTableModel: viewModel,
              isSavedView,
              scope,
              schema,
            };
        
            return (
              <div className={getClassName({ descendantName: 'mainContent' })}>
                <UserStoriesDataTable {...userStoriesDataTableProps} />
              </div>
            );
          }
        }
        
        ```
1. Registering the Page
    -  Import the `index.js` in the root of your page directory in `~/projects/churro/src/viewport/pageModules.js` and add it to the list of registered page modules.
        - Note: Where you add your page in the list is only important if you are trying to add a new replacement for an existing page. In this case, it should be enough to make sure that your page is registered __above__ `LegacyPage`.
        - Note: You may need to restart your Burro dev server after making this change. 
1. If Applicable, turn on your feature toggle by logging into your environment as the "god mode" user. For TestN/UEShell this user will be `rallyadmin`. If you're developing against a local stack, then it will be `slmadmin`. Ask your nearest dev or QA for the password.
1. Navigate to your page in the application and view it in all it's glory!
