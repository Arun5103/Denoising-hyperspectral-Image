# Hyperspectral Image Denoising and Segmentation

## Overview
This project focuses on denoising and segmenting hyperspectral images using convolutional neural networks (CNNs). Hyperspectral imaging captures images at multiple wavelengths, providing detailed spectral information for each pixel. The goal is to denoise these images and then perform segmentation to classify different regions based on their spectral signatures.

## Requirements
- Python 3.x
- PyTorch
- NumPy
- SciPy
- Matplotlib
- Scikit-learn

## Directory Structure
```
/DATASET_UAS
├── IMAGES
│   ├── b1-9944.mat
│   ├── b1-9945.mat
│   └── ... (more images)
└── MASKS
    ├── b1-9944_mask.mat
    ├── b1-9945_mask.mat
    └── ... (more masks)
```

## Dataset
The dataset consists of hyperspectral images and their corresponding segmentation masks. Each image is stored in a `.mat` file with a key for accessing the data.

- Images: `DATASET_UAS/IMAGES/`
- Masks: `DATASET_UAS/MASKS/`

## Model Architecture
### DenoisingCNN
A Convolutional Neural Network for denoising hyperspectral images. The network consists of an encoder-decoder architecture with convolutional and transpose convolutional layers.

### SegmentationCNN
A Convolutional Neural Network for segmenting hyperspectral images. The network also uses an encoder-decoder architecture similar to the DenoisingCNN but the final layer produces class labels.

## Usage
### Dataset Class
A custom PyTorch Dataset class to load hyperspectral images and their corresponding masks.
```python
class HyperspectralDataset(Dataset):
    def __init__(self, images_dir, masks_dir):
        self.images_dir = images_dir
        self.masks_dir = masks_dir
        self.image_files = [f for f in os.listdir(images_dir) if f.endswith('.mat')]
        self.mask_files = [f for f in os.listdir(masks_dir) if f.endswith('.mat')]
        
    def __len__(self):
        return len(self.image_files)
    
    def __getitem__(self, idx):
        image_file = self.image_files[idx]
        filename = os.path.splitext(image_file)[0]
        image_path = os.path.join(self.images_dir, image_file)
        image_data = loadmat(image_path)
        image = image_data[filename]

        mask_file = self.mask_files[idx]
        mask_path = os.path.join(self.masks_dir, mask_file)
        mask_data = loadmat(mask_path)
        mask = mask_data['image']

        image = image.astype(np.float32)
        image = image / np.max(image)

        image = torch.tensor(image).permute(2, 0, 1)
        mask = torch.tensor(mask).permute(2, 0, 1)

        return image, mask
```

### Model Training
Training the DenoisingCNN model with the hyperspectral dataset.
```python
model = DenoisingCNN().to(device)
criterion = nn.MSELoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
scheduler = torch.optim.lr_scheduler.ReduceLROnPlateau(optimizer, 'min', patience=3, factor=0.5)

num_epochs = 10
best_val_loss = float('inf')
patience = 10
counter = 0

for epoch in range(num_epochs):
    model.train()
    running_loss = 0.0
    for images, masks in dataloader:
        if images is None or masks is None:
            continue

        images = images.to(device)
        masks = masks.to(device)

        outputs = model(images)
        loss = criterion(outputs, images)

        optimizer.zero_grad()
        loss.backward()
        optimizer.step()

        running_loss += loss.item()
    
    avg_loss = running_loss / len(dataloader)
    print(f'Epoch [{epoch+1}/{num_epochs}], Loss: {avg_loss:.4f}')

    model.eval()
    with torch.no_grad():
        validation_loss = 0.0
        for images, masks in dataloader:
            if images is None or masks is None:
                continue
            images = images.to(device)
            outputs = model(images)
            loss = criterion(outputs, images)
            validation_loss += loss.item()
        avg_val_loss = validation_loss / len(dataloader)
        print(f'Validation Loss: {avg_val_loss:.4f}')

    scheduler.step(avg_val_loss)

    if avg_val_loss < best_val_loss:
        best_val_loss = avg_val_loss
        counter = 0
        torch.save(model.state_dict(), 'best_model.pth')
    else:
        counter += 1
        if counter >= patience:
            print("Early stopping")
            break
```

### Evaluation and Visualization
Loading the best model and visualizing the results.
```python
def segment_image(denoised_image, num_segments=5):
    flat_image = denoised_image.reshape(-1, denoised_image.shape[-1])
    kmeans = KMeans(n_clusters=num_segments, random_state=42)
    labels = kmeans.fit_predict(flat_image)
    segmented_image = labels.reshape(denoised_image.shape[:-1])
    return segmented_image

def visualize_results(original, denoised, segmented):
    fig, axs = plt.subplots(1, 3, figsize=(15, 5))
    axs[0].imshow(original[:,:,:3])
    axs[0].set_title('Original')
    axs[0].axis('off')
    axs[1].imshow(denoised[:,:,:3])
    axs[1].set_title('Denoised')
    axs[1].axis('off')
    axs[2].imshow(segmented, cmap='viridis')
    axs[2].set_title('Segmented')
    axs[2].axis('off')
    plt.tight_layout()
    plt.show()

model.load_state_dict(torch.load('best_model.pth'))
model.eval()

with torch.no_grad():
    for images, _ in dataloader:
        images = images.to(device)
        denoised_images = model(images)
        
        original_image = images[0].cpu().numpy().transpose(1, 2, 0)
        denoised_image = denoised_images[0].cpu().numpy().transpose(1, 2, 0)
        
        segmented_image = segment_image(denoised_image)
        
        visualize_results(original_image, denoised_image, segmented_image)
        break
```

## Conclusion
This project demonstrates the process of denoising and segmenting hyperspectral images using deep learning techniques. By utilizing CNNs, we can effectively clean and classify different regions within these high-dimensional images.
