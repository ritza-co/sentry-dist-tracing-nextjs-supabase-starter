# Note-Taking App

A basic Next.js application for creating notes. Built with TypeScript and Supabase.

## Features

- User authentication (sign up and sign in)
- Notes creation
- Supabase PostgreSQL database
- Light and dark modes
- Styling with [Tailwind CSS](https://tailwindcss.com)
- Components with [shadcn/ui](https://ui.shadcn.com/)

## Install Dependencies

Install the dependencies using the following command:

```bash
npm install
```

## Set Up a Supabase Database

[Sign up to Supabase](https://supabase.com/dashboard/sign-up) if you don't already have an account. Create an organization:

![Create Supabase org](./app/assets/images/supabase-create-org.png)

Create a new project:

![Create Supabase project](./app/assets/images/supabase-create-project.png)


From the left-hand navigation, open the **Table Editor** page and click **+ New Table**. Create a table called "notes" with the following columns:

| Name    | Type | Default Value | Primary | Is Identity |
| ------- | ---- | ------------- | ------- | ----------- |
| id      | int8 | null          | true    | true        |
| title   | text | null          | false   |             |
| content | text | null          | false   |             |
| user_id | uuid | null          | false   |             |

![Create notes table](./app/assets/images/supabase-create-table.png)

Click **Add foreign key relation** at the bottom of the form. Select the **users** table in the Supabase **Auth** schema to reference to. Create a one-to-one relationship between `notes.user_id` and `auth.users.id`. Click **Save**:

![Create foreign key](./app/assets/images/supbase-create-foreign-key.png)

You'll see an empty table:

![Empty notes table](./app/assets/images/supabase-empty-table.png)

From the left-hand navigation menu, open the **SQL Editor** page and add the following SQL queries to the editor:

```sql
-- Create policy for SELECT operations
-- Users can only view their own notes
CREATE POLICY "Users can view their own notes" 
ON notes
FOR SELECT 
TO authenticated
USING ((select auth.uid()) = user_id);

-- Create policy for INSERT operations
-- Users can only create notes for themselves
CREATE POLICY "Users can create their own notes" 
ON notes
FOR INSERT 
TO authenticated
WITH CHECK ((select auth.uid()) = user_id);

-- Create policy for UPDATE operations
-- Users can only update their own notes
CREATE POLICY "Users can update their own notes" 
ON notes
FOR UPDATE 
TO authenticated
USING ((select auth.uid()) = user_id);

-- Create policy for DELETE operations
-- Users can only delete their own notes
CREATE POLICY "Users can delete their own notes" 
ON notes
FOR DELETE 
TO authenticated
USING ((select auth.uid()) = user_id);
```

This query creates [policies](https://supabase.com/docs/guides/database/postgres/row-level-security#creating-policies) for the notes table that restrict CRUD actions on notes to the note's owner. This works because [Row Level Security](https://supabase.com/docs/guides/database/postgres/row-level-security) was enabled when the table was created, which is recommended. 

Click the **Run** button at the lower right of the page to run the query. You should see "Success. No rows returned" printed to the **Results** tab. 

Now add the following SQL query to the editor:

```sql
-- Create a stored procedure for slow retrieval of notes
CREATE OR REPLACE FUNCTION slow_get_notes(
  p_user_id UUID
) 
RETURNS SETOF notes AS $$
BEGIN
  -- Sleep for 120 seconds
  PERFORM pg_sleep(120);
  
  -- Then return the notes for the user
  RETURN QUERY
  SELECT * FROM notes
  WHERE user_id = p_user_id;
END;
$$ LANGUAGE plpgsql;
```

This query creates a [Postgres function](https://supabase.com/docs/guides/database/functions) called `slow_get_notes` that fetches the user's notes. A 120-second delay is added to the query using the Postgres `pg_sleep` function. 

Click the **Run** button at the lower right of the page to run the query. You should see "Success. No rows returned" printed to the **Results** tab. 
 
 ## Connecting Your Supabase Project to the Next.js Note-Taking App

In the Next.js note-taking app, create a `.env` file in the root of the project and add the following variables to it:

```
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY
```

Open [your Supabase project's API settings](https://app.supabase.com/project/_/settings/api) and add the project URL and the `anon` and `public` API key values to the `.env` file.

## Creating Example Notes

Run the local development server:

```bash
npm run dev
```

You'll see the note-taking app home page:

![Notes app](./app/assets/images/notes-app.png)

Sign up, verify your email address, then log in to see the notes page:

![Notes app - logged in](./app/assets/images/notes-app-logged-in.png)

Create some example notes:

![Notes app - with notes](./app/assets/images/notes-app-with-notes.png)

## Updating the `getNotes` Server Action to Use the Postgres Function 

In the `/app/actions.ts` file, replace the `getNotes` Server Action with the following `getNotes` function:
 
```ts
export async function getNotes() {
  "use server";
  try {

  const supabase = await createClient();
  
  // Get the current user
  const { data: userData } = await supabase.auth.getUser();
  if (!userData.user) {
    throw new Error("User not authenticated");
  }
  
  const { data: notes, error } = await supabase.rpc(
    'slow_get_notes',
    { p_user_id: userData.user.id }
  );
  
  // If there's a database error, throw it
  if (error) {
    console.error('Error fetching notes:', error);
    throw new Error("Failed to retrieve notes");
  }
  
  // Return the notes
  return notes;

  } catch (error) {
    console.error('Error in getNotes:', error);
    
    // Create a custom error with a generic message
    const clientError = new Error("Something went wrong while loading your notes. Please try again later.");
    
    // Rethrow the error with generic message for the client
    throw clientError;
  }
}
```

This update changes the `getNotes` Server Action to use the `slow_get_notes` Postgres function we created, calling it with a remote procedure call (RPC) to fetch the user's notes. 

## Deploy With Vercel

If you haven't already, [sign up to Vercel](https://vercel.com/signup) using your GitHub account. Save this note-taking project in a GitHub repo, then click [**New Project**](https://vercel.com/new) at the top right of your dashboard. You'll be presented with a list of Git repositories that the Git account you signed up with has write access to. Import the notes project, and a page will be displayed where you can configure your project before it's deployed:

- Select Next.js as the [**Framework Preset**](https://vercel.com/docs/deployments/configure-a-build#framework-preset).
- Keep the **Root Directory** as `./`.
- Add the `.env` variables to [**Environment Variables**](https://vercel.com/docs/environment-variables). 
- Click **Deploy**.

