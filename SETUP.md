# Quick Setup Guide

## Step 1: Create Supabase Project

1. Go to [supabase.com](https://supabase.com)
2. Sign up or log in
3. Click "New Project"
4. Fill in:
   - **Name**: Range Ranks (or any name)
   - **Database Password**: Choose a strong password (save it!)
   - **Region**: Choose closest to you
5. Wait ~2 minutes for project to be created

## Step 2: Set Up Database

1. In your Supabase project, go to **SQL Editor** (left sidebar)
2. Click **New Query**
3. Open `supabase/migrations/001_initial_schema.sql` from this project
4. Copy the entire contents
5. Paste into SQL Editor
6. Click **Run** (or press Cmd/Ctrl + Enter)
7. You should see "Success. No rows returned"

## Step 3: Get API Credentials

1. In Supabase, go to **Settings** (gear icon) → **API**
2. Copy these two values:
   - **Project URL** (looks like: `https://xxxxx.supabase.co`)
   - **anon public** key (long string starting with `eyJ...`)

## Step 4: Configure Local Environment

1. Create a `.env` file in the project root:
   ```bash
   cp .env.example .env
   ```

2. Edit `.env` and paste your credentials:
   ```
   VITE_SUPABASE_URL=https://xxxxx.supabase.co
   VITE_SUPABASE_ANON_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
   ```

## Step 5: Install and Run

```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

The app should open in your browser at `http://localhost:3000`

## Step 6: Test It Out

1. Click "Sign Up" to create an account
2. Log in with your credentials
3. Log a shooting session
4. Check the leaderboard!

## Troubleshooting

### "Invalid API key"
- Double-check your `.env` file has the correct values
- Make sure there are no extra spaces or quotes
- Restart the dev server after changing `.env`

### "Table doesn't exist"
- Go back to SQL Editor and verify the migration ran successfully
- Check that you see tables: `profiles`, `shooting_sessions`, `follows` in the Table Editor

### Can't sign up
- Check Supabase **Authentication** → **Settings**
- Make sure "Enable email signup" is ON
- For testing, you can disable "Confirm email" temporarily

### Real-time not working
- Go to **Database** → **Replication** in Supabase
- Enable replication for `shooting_sessions` table
- Or check that RLS policies are set correctly

## Next Steps

- Deploy to Vercel/Netlify (see README.md)
- Customize the drills in `src/app.js`
- Add more features!

