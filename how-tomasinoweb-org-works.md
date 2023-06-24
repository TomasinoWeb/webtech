What follows is a system design overview of [tomasinoweb.org](https://tomasinoweb.org), the official website of the premier digital media organization of the University of Santo Tomas.

The website is a [Node](https://nodejs.org/en) application running an [Express](https://expressjs.com/) HTTP server. The frontend is served by a [Next.js](https://nextjs.org/) application that renders [React](https://react.dev) on the server side. The backend is written with a [custom REST API framework](https://www.npmjs.com/package/@scinorandex/rpscin) that allows for typesafe API calls between the frontend and the backend. The main database *(for now)* is a single SQLite database, however this can be easily swapped for a PostgreSQL database since the backend is using [Prisma](https://www.prisma.io/) as an ORM. The entire codebase is strictly typed using [TypeScript](https://www.typescriptlang.org/).

The backend is connected to the express server under `/api` and the Next.js server is connected under `/`. Aside from the afformentioned subroutes, a static file server is also connected under `/static` to serve dynamic content in development. In production, the entire application is reverse proxied through NGINX, which also acts as a reverse proxy for [Draft 143](https://draft143.com).

This stack was chosen as it allowed for fairly rapid development, with the website having being launched 67 days after the initial commit in the repo. A major factor that went into the speed of development was the custom framework that I made, that took care of API request body validation, file uploads, made middleware authentication easier, etc., without the cost of runtime performance. 

## Backend

`@scinorandex/rpscin` has no runtime costs as it is just a wrapper over Express that captures the input and return types of API endpoints. With these types, you can create an API client in the frontend that enforces request input types (being able to enforce body, query parameter, and path parameters) and knows the structure of the returned data for each API endpoint. This library also allows for middleware support which makes actions like CSRF protection, authentication, and role based acess control (RBAC) easier.

Each endpoint can be defined from a "procedure", which is a collection of middleware tied together. A procedure can be constructed with a middleware that checks if the user is authenticated, and any endpoint defined from said procedure would have access to the user's database object. By abusing TypeScript, new API endpoints could be constructed with ease, and be robust against trivial type errors.

### Request Validation

Request body validation is performed by [Zod](https://zod.dev/), a parsing + validation library. Zod is library that validates values against a specified schema, in this case it is used to validate the request body and query parameters against a schema for an API endpoint. Every procedure has a `.input()` and `.query()` method that accepts a Zod validator, which validates the request body and query parameters respectively, automatically throwing an error if it fails to parse. The validated inputs are then available to the endpoint through the locals object (passed as a parameter), as the `input` and `query` properties respectively. 

Here's an example of GET and POST methods under `/api/posts/gallery` using the `.query()` and `.input()` methods:

```ts
export const galleryRouter = postRouter.subroute("/gallery").config({
  "/": {
    get: baseProcedure
      .query(
        z.object({
          take: z.number().max(25).default(25),
          skip: z.number().optional(),
          type: GalleryPostTypeEnum.validator.optional(),
        })
      )
      .use(async (req, res, { query }) => {
        const galleryPosts = await db.galleryPost.findMany({
          take: query.take,
          skip: query.skip == null ? undefined : query.skip,
          where: query.type !== undefined ? { type: query.type } : undefined,
          orderBy: { createdAt: "desc" },
          include: { mainImage: true },
        });

        return { galleryPosts: galleryPosts.map(Cleanse.galleryPost) };
      }),

    post: authProcedure
      .input(
        z.object({
          credits: z.string(),
          link: z.string(),
          title: z.string(),
          mainImageCaption: z.string(),
          type: GalleryPostTypeEnum.validator,
          mainImageUuid: z.string(),
          excerpt: z.string(),
        })
      )
      .use(async (req, res, { input }) => {
        const newGalleryPost = await db.galleryPost.create({
          data: {
            credits: input.credits,
            excerpt: input.excerpt,
            link: input.link,
            title: input.title,
            mainImageCaption: input.mainImageCaption,
            type: input.type,
            mainImageUuid: input.mainImageUuid,
          },
          include: { mainImage: true },
        });

        return Cleanse.galleryPost(newGalleryPost);
      }),
  },
})
```

### Scheduled Posts

Scheduled posts are handled by [node-scheduler](https://www.npmjs.com/package/node-schedule), with a custom class for keeping track of active jobs. The class is generic and accepts a callback in the constructor which is called whenever a job should be executed, with a unique identifier passed as a parameter. Scheduled posts are also kept track of in the database as a seperate table so it can be loaded whenever the server restarts.

The following is a shortened version of the class as well as the post scheduler itself:

```ts
import schedule from "node-schedule";

export class Scheduler<T> {
  private jobs: Map<T, schedule.Job> = new Map();
  public constructor(private readonly callback: (unique_identifier: T) => Promise<void>) {}

  schedule(date: Date, uuid: T) {
    /** Don't add any jobs set in the past */
    if (date.valueOf() < Date.now()) return;

    /** Cancel any existing jobs with the same uuid */
    this.cancel(uuid);

    this.jobs.set(uuid, schedule.scheduleJob(date, () => {
      this.callback(uuid).catch(console.warn);
    }));
  }

  cancel(uuid: T) {
    const oldJob = this.jobs.get(uuid);
    if (oldJob) {
      oldJob.cancel();
      this.jobs.delete(uuid);
    }
  }
}

/** The callback is executed whenever a post should be published */
const postScheduler = new Scheduler<string>(async (postUuid) => {
  const updatedPost = await db.post.update({
    where: { uuid: postUuid },
    data: { published: PublishedEnum.enums.immediately },
    include: CleansePostIncludeOptions,
  });

  await db.scheduledPost.delete({ where: { postUuid } });
});

/**
 * Example usage: 
 * postScheduler.schedule(new Date("2023-07-12"), post.uuid);
 * /
```

### Search

Although prisma is used to interface with the primary database, prisma for SQLite [does not support full text search](https://github.com/prisma/prisma/issues/9414). Thus, another database has to be maintained which is solely used for fulltext search capabilities. A script is provided which initializes the search database, and the server keeps the search database updated every 5 minutes by searching for posts that have been created / updated in the primary database recently.

## Frontend

The frontend is writtein in React, using SCSS modules for styling, and server side rendered by Next.js. It is split up into the admin-side and the user-side.

The admin side and user side use different layouts and require different pieces of data to work, thus a system for working with different data had to be made.

### Layouts

It had to meet the following requirements
 - It allows for pages to load their needed data in the server side
 - The layout could load the server side data that it needs
 - The page could manipulate the layout in the client side

Thus the following system was made:

```tsx
// The data that the layout needs that must be fetched from the server side
export interface InternalProps {}

// Fetch the data that the layout needs server side
async function fetchInternalProps(ctx: GetServerSidePropsContext): Promise<InternalProps> {
	return {}
}

export function PageProps<PropType extends { [key: string]: any }>(
  // passthrough is a function that returns the data that the page itself needs from server side
  passthrough: (context: GetServerSidePropsContext) => Promise<GetServerSidePropsResult<PropType>>
){
  // return the function exported as getServerSideProps in the page file
  return async function(context: GetServerSidePropsContext): Promise<GetServerSidePropsResult<
    { serverSideProps: PropType; internalProps: PublicInternalProps }
  >>{
    const passthroughResults = await passthrough(context);
    
    if("props" in passthroughResults){
      /**
       * The passthrough successfully ran and returned the page's
       * serverSideProps, which we take and inject the internalProps onto
       */
      const serverSideProps = await passthroughResults.props;
      const internalProps = await fetchInternalProps(contxet);
      return { props: { serverSideProps, internalProps } }
    }else{
      /** 
       * Something wrong happened in the passthrough function so return it's result
       * It could be that the required resource for a page is not found
       * Or a redirect is requested for a different page
       */ 
      return passthroughResult;
    }
  }
}

/** Props from InternalProps that should also be passed to the page handler */
interface ExportedInternalProps {
}

/** Props returned from the page handler, which must include the children of the page by default */
interface LayoutProps{
  children: React.ReactNode;
}

export const InternalPropsProvider = React.createContext<InternalProps>(null as any);

export function Page<PropType extends { [key: string]: any }>(
  handler: (props: PropType & ExportedInternalProps) => LayoutProps
) {
  /**
   * Returns the default export of a page, which receives serverSideProps
   * and internalProps from the PageProps function above 
   */
  return (props: { serverSideProps: PropType; internalProps: InternalProps }) => {
    const exportedInternalProps: ExportedInternalProps = {};

    return (
      <InternalPropsProvider.Provider value={props.internalProps}>
        {(() => {
          /** 
           * We need to call handler inside an IIFE so that it consumers of
           * InternalPropsProvider have the proper context instead of accessing null.
           * We pass the server side props that it requested and also selected internal props.
           */
          const layoutProps = handler({ ...props.serverSideProps, ...exportedInternalProps});
          return <Layout {...layoutProps} {...props.internalProps} />;
        })()}
      </InternalPropsProvider.Provider>
    );
  };
}

function Layout(props: LayoutProps & InternalProps){
  // LayoutProps could be used to manipulate parts of the UI from the page handler code
  return <main>
    {props.children}
  </main>
}
```

With the setup above, successful results from the passthrough function could also be cached to not have to reduce calls. This requires a hashing function that returns a hashstring for a request object. This hashstring could be taken from URL parameters.
