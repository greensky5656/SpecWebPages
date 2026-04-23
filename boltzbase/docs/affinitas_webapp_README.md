# Affinitas

Affinitas is a modern web application for biomolecular binding prediction, featuring an intuitive interface for configuring and managing protein-ligand binding predictions using the Boltz deep learning models.

## Features

- **Intuitive Configuration Interface**: Web-based form for setting up binding prediction parameters
- **Real-time Run Management**: Track prediction jobs from submission to completion
- **3D Structure Visualization**: Interactive molecular viewer powered by Molstar
- **Modern Dark Mode UI**: Clean, professional interface with dark theme support
- **Run History**: Searchable database of all prediction runs
- **Live Status Updates**: Monitor running jobs in real-time
- **Download Results**: Export complete prediction results

## Installation

### Prerequisites

- macOS with Homebrew
- Python 3.12 (required)
- Node.js 18+ (for the web UI)
- CUDA-capable GPU (recommended)

### Initial Setup (macOS only)

Before running the app, you must set up Python 3.12 using Homebrew:

```bash
# Setup Homebrew environment
eval "$(/opt/homebrew/bin/brew shellenv)"

# Update Homebrew
brew update

# Install Python 3.12
brew install python@3.12

# Add Python 3.12 to your PATH
echo 'export PATH="/opt/homebrew/opt/python@3.12/bin:$PATH"' >> ~/.zshrc

# Reload shell configuration
exec zsh -l

# Verify Python version (should print "Python 3.12.x")
python3.12 -V
```

### Backend Setup

1. Clone the repository:
```bash
git clone https://github.com/greensky5656/boltzbase.git
cd boltz
```

2. Create a virtual environment with Python 3.12:
```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

3. Install dependencies:
```bash
pip install -e .
```

4. **Verify installation** (IMPORTANT):
```bash
# Check that boltz command exists
which boltz
# Should output: /path/to/boltz/.venv/bin/boltz

# Check boltz is working
boltz --help
# Should show help text
```

For CUDA support:
```bash
pip install -e ".[cuda]"
```

### Frontend Setup

1. Navigate to the UI directory:
```bash
cd boltz-ui
```

2. Install dependencies:
```bash
npm install
```

3. Ensure the lib directory exists (should be created automatically):
```bash
ls boltz-ui/lib/
# Should show: db.ts and boltz-client.ts
```

4. Start the development server:
```bash
npm run dev
```

The web interface will be available at `http://localhost:3000`

**Note**: If you encounter module resolution errors like "Can't resolve '@/lib/db'", try:

1. First, check if the lib directory exists (from the boltz-ui directory):
```bash
cd boltz-ui
ls lib/
# Should show: db.ts and boltz-client.ts
# Note: DO NOT use /lib (leading slash) - that's the system directory!
```

3. If still missing, clean and reinstall:
```bash
cd boltz-ui
rm -rf node_modules .next
npm install
npm run dev
```

4. If the lib directory files are missing from git, you may need to copy them from another machine or recreate them.

## Usage

### Web Interface

1. **Open the application**: Navigate to `http://localhost:3000` in your browser
2. **Configure a Run**: 
   - Go to the "Configurations" page
   - Enter your protein sequence and ligand SMILES
   - Adjust prediction parameters as needed
   - Click "Launch Process" to start the prediction
3. **View Results**:
   - Navigate to "Runs & Results" page
   - Select a completed run from the sidebar
   - View the 3D structure in the molecular viewer
   - Expand sections to see key metrics and full output
   - Download results using the "Download Results" button

### Configuration Parameters

The web interface allows you to configure various prediction parameters:

- **Protein Input**: Sequence or structure file
- **Ligand Input**: SMILES string or structure file
- **MSA Settings**: Multiple sequence alignment options
- **Model Selection**: Choose between Boltz1 and Boltz2 models
- **Output Options**: Control result format and detail level

## Project Structure

```
boltz/
├── boltz-ui/              # Next.js web application
│   ├── app/               # Next.js app directory
│   │   ├── page.tsx       # Configuration page
│   │   ├── runs/          # Runs & Results page
│   │   └── api/           # API routes
│   ├── components/        # React components
│   │   └── Header.tsx     # Navigation header
│   ├── public/            # Static assets
│   └── package.json       # Frontend dependencies
├── src/boltz/             # Backend Boltz CLI
│   ├── main.py            # CLI entry point
│   ├── data/              # Data processing
│   └── model/             # Neural network models
├── examples/              # Example configuration files
└── pyproject.toml         # Python dependencies
```

## Technology Stack

### Frontend
- **Next.js 16**: React framework with App Router
- **TypeScript**: Type-safe JavaScript
- **Tailwind CSS**: Utility-first CSS framework
- **Molstar**: 3D molecular visualization
- **SQLite**: Local database for run history
- **Drizzle ORM**: Type-safe database access

### Backend
- **Python 3.12**: Core language
- **PyTorch**: Deep learning framework
- **Boltz Models**: Pre-trained neural networks for structure prediction
- **Hydra**: Configuration management

## Development

### Running the Development Server

```bash
cd boltz-ui
npm run dev
```

### Running Tests

```bash
# Frontend tests
cd boltz-ui
npm test

# Backend tests
pytest tests/
```

### Building for Production

```bash
# Frontend build
cd boltz-ui
npm run build
npm start

# Backend install
pip install -e .
```

## API Endpoints

- `POST /api/runs/new` - Create a new prediction run
- `GET /api/runs` - List all runs
- `GET /api/runs/[id]` - Get run details
- `GET /api/files/[...path]` - Serve result files
- `GET /api/run-history` - Download run history CSV

## Models

Affinitas uses the Boltz deep learning models:

### Boltz1
- Focus on protein structure confidence prediction
- Best for: High-confidence structure generation

### Boltz2
- Enhanced version with improved accuracy
- Includes specialized affinity prediction model
- Best for: Binding affinity and complex structure prediction

## License

See LICENSE file for details.

## Contributing

Contributions are welcome! Please feel free to submit pull requests or open issues for bugs and feature requests.

## Support

For questions and support, please open an issue on GitHub.

## Acknowledgments

- PyTorch Lightning for training infrastructure
- Molstar for 3D molecular visualization
- RDKit for molecular processing
- Next.js for the web framework