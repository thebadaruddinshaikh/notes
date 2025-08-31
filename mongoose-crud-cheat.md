# Mongoose CRUD Operations Cheat Sheet

## 🚀 Quick Setup

```bash
npm install mongoose express dotenv cors joi bcryptjs
```

```javascript
// Basic connection
const mongoose = require('mongoose');
await mongoose.connect(process.env.MONGODB_URI, {
  useUnifiedTopology: true,
  useNewUrlParser: true
});
```

## 📋 Schema Essentials

### Basic Schema Structure
```javascript
const userSchema = new mongoose.Schema({
  // Basic types
  name: { type: String, required: true, trim: true },
  email: { type: String, unique: true, lowercase: true },
  age: { type: Number, min: 0, max: 120 },
  isActive: { type: Boolean, default: true },
  
  // Complex types
  tags: [String], // Array of strings
  metadata: mongoose.Schema.Types.Mixed, // Any type
  userId: mongoose.Schema.Types.ObjectId, // ObjectId
  
  // Nested objects
  profile: {
    firstName: String,
    lastName: String,
    avatar: { type: String, default: 'default.png' }
  },
  
  // Array of objects
  addresses: [{
    street: { type: String, required: true },
    city: String,
    isDefault: { type: Boolean, default: false }
  }],
  
  // References
  posts: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Post' }]
}, {
  timestamps: true, // Adds createdAt, updatedAt
  toJSON: { virtuals: true }
});
```

### Schema Features
```javascript
// Virtual fields (computed properties)
userSchema.virtual('fullName').get(function() {
  return `${this.profile.firstName} ${this.profile.lastName}`;
});

// Pre-save middleware
userSchema.pre('save', async function(next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 12);
  }
  next();
});

// Instance methods
userSchema.methods.comparePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

// Static methods
userSchema.statics.findByEmail = function(email) {
  return this.findOne({ email });
};

// Indexes for performance
userSchema.index({ email: 1 });
userSchema.index({ 'profile.firstName': 1, 'profile.lastName': 1 });
```

## 🔧 CREATE Operations

### Basic Create
```javascript
// Single document
const user = new User({ name: 'John', email: 'john@example.com' });
await user.save();

// Or using create()
const user = await User.create({ name: 'John', email: 'john@example.com' });
```

### Bulk Create
```javascript
// Insert many
const users = await User.insertMany([
  { name: 'John', email: 'john@example.com' },
  { name: 'Jane', email: 'jane@example.com' }
]);

// With transaction
const session = await mongoose.startSession();
session.startTransaction();
try {
  await User.insertMany(userData, { session });
  await session.commitTransaction();
} catch (error) {
  await session.abortTransaction();
  throw error;
} finally {
  session.endSession();
}
```

## 📖 READ Operations

### Basic Queries
```javascript
// Find all
const users = await User.find();

// Find with conditions
const activeUsers = await User.find({ isActive: true });

// Find one
const user = await User.findById(id);
const user = await User.findOne({ email: 'john@example.com' });
```

### Advanced Queries
```javascript
// Comparison operators
User.find({ age: { $gte: 18, $lt: 65 } }) // age >= 18 and age < 65
User.find({ status: { $in: ['active', 'pending'] } }) // status in array
User.find({ name: { $regex: 'john', $options: 'i' } }) // case-insensitive regex

// Logical operators
User.find({
  $or: [
    { email: 'john@example.com' },
    { username: 'john123' }
  ]
})

User.find({
  $and: [
    { age: { $gte: 18 } },
    { isActive: true }
  ]
})
```

### Query Modifiers
```javascript
// Projection (select specific fields)
User.find({}, 'name email') // Only name and email
User.find({}).select('name email -_id') // Include name, email, exclude _id
User.find({}).select('-password') // Exclude password

// Sorting
User.find({}).sort({ createdAt: -1 }) // Newest first
User.find({}).sort('name -age') // Name ascending, age descending

// Pagination
const page = 2, limit = 10;
User.find({})
  .skip((page - 1) * limit)
  .limit(limit)

// Population (join with referenced collections)
User.findById(id).populate('posts')
User.find({}).populate('posts', 'title content') // Only specific fields
User.find({}).populate({
  path: 'posts',
  match: { isPublished: true },
  options: { sort: { createdAt: -1 } }
})
```

### Aggregation Pipeline
```javascript
// Group and count
User.aggregate([
  { $group: { _id: '$status', count: { $sum: 1 } } },
  { $sort: { count: -1 } }
])

// Complex aggregation
User.aggregate([
  { $match: { isActive: true } }, // Filter
  { $unwind: '$skills' }, // Flatten array
  { $group: { // Group by skill
    _id: '$skills.name',
    userCount: { $sum: 1 },
    avgLevel: { $avg: '$skills.level' }
  }},
  { $sort: { userCount: -1 } },
  { $limit: 10 }
])
```

## ✏️ UPDATE Operations

### Basic Updates
```javascript
// Update one document
await User.findByIdAndUpdate(id, { name: 'New Name' }, { new: true });
await User.updateOne({ _id: id }, { $set: { name: 'New Name' } });

// Update many
await User.updateMany({ isActive: false }, { $set: { status: 'inactive' } });
```

### Update Operators
```javascript
// $set - Set field values
User.updateOne({ _id: id }, { $set: { 'profile.firstName': 'John' } })

// $unset - Remove fields
User.updateOne({ _id: id }, { $unset: { temporaryField: 1 } })

// $inc - Increment/decrement
User.updateOne({ _id: id }, { $inc: { loginCount: 1, score: -5 } })

// Array operations
User.updateOne({ _id: id }, { $push: { tags: 'newTag' } }) // Add to array
User.updateOne({ _id: id }, { $addToSet: { tags: 'uniqueTag' } }) // Add if unique
User.updateOne({ _id: id }, { $pull: { tags: 'oldTag' } }) // Remove from array
User.updateOne({ _id: id }, { $pop: { tags: 1 } }) // Remove last element (or -1 for first)

// Update array elements
User.updateOne(
  { _id: id, 'addresses.type': 'home' },
  { $set: { 'addresses.$.isDefault': true } }
) // Update first matching array element

// Update all array elements
User.updateOne({ _id: id }, { $set: { 'addresses.$[].isVerified': false } })

// Update with conditions
User.updateOne(
  { _id: id },
  { $set: { 'addresses.$[elem].isDefault': true } },
  { arrayFilters: [{ 'elem.type': 'home' }] }
)
```

### Upsert Operations
```javascript
// Create if doesn't exist, update if exists
await User.updateOne(
  { email: 'john@example.com' },
  { $set: { lastLogin: new Date() } },
  { upsert: true }
);
```

## 🗑️ DELETE Operations

### Basic Deletes
```javascript
// Delete one
await User.findByIdAndDelete(id);
await User.deleteOne({ _id: id });

// Delete many
await User.deleteMany({ isActive: false });

// Find and delete (returns deleted document)
const deletedUser = await User.findByIdAndDelete(id);
```

### Soft Delete Pattern
```javascript
// Mark as deleted instead of actual deletion
await User.updateOne({ _id: id }, { 
  $set: { 
    isDeleted: true, 
    deletedAt: new Date() 
  } 
});

// Find non-deleted documents
User.find({ isDeleted: { $ne: true } })
```

## 🔍 Query Performance Tips

### Indexing
```javascript
// Single field index
userSchema.index({ email: 1 });

// Compound index
userSchema.index({ status: 1, createdAt: -1 });

// Text index for search
userSchema.index({ name: 'text', bio: 'text' });

// Sparse index (only for documents with the field)
userSchema.index({ phoneNumber: 1 }, { sparse: true });

// Unique index
userSchema.index({ email: 1 }, { unique: true });
```

### Query Optimization
```javascript
// Use explain() to analyze queries
User.find({ status: 'active' }).explain('executionStats')

// Limit fields in projections
User.find({}, 'name email') // Faster than returning all fields

// Use lean() for read-only operations (returns plain JS objects)
User.find({}).lean() // Much faster, but no Mongoose methods

// Batch operations
User.find({}).cursor().eachAsync(async (doc) => {
  // Process each document
}, { parallel: 10 })
```

## 🛡️ Validation & Error Handling

### Built-in Validators
```javascript
const schema = new mongoose.Schema({
  email: {
    type: String,
    required: [true, 'Email is required'],
    unique: true,
    validate: {
      validator: function(v) {
        return /^\w+([.-]?\w+)*@\w+([.-]?\w+)*(\.\w{2,3})+$/.test(v);
      },
      message: 'Invalid email format'
    }
  },
  age: {
    type: Number,
    min: [0, 'Age must be positive'],
    max: [120, 'Age must be realistic']
  },
  status: {
    type: String,
    enum: {
      values: ['active', 'inactive', 'pending'],
      message: 'Status must be active, inactive, or pending'
    }
  }
});
```

### Error Handling Patterns
```javascript
try {
  const user = await User.create(userData);
  res.json({ success: true, data: user });
} catch (error) {
  if (error.name === 'ValidationError') {
    const errors = Object.values(error.errors).map(e => e.message);
    return res.status(400).json({ success: false, errors });
  }
  
  if (error.code === 11000) { // Duplicate key error
    return res.status(409).json({ 
      success: false, 
      message: 'Email already exists' 
    });
  }
  
  res.status(500).json({ 
    success: false, 
    message: 'Internal server error' 
  });
}
```

## 🔧 Common Patterns & Best Practices

### Pagination Helper
```javascript
const paginate = async (Model, query = {}, options = {}) => {
  const page = parseInt(options.page) || 1;
  const limit = parseInt(options.limit) || 10;
  const skip = (page - 1) * limit;
  
  const [data, total] = await Promise.all([
    Model.find(query).skip(skip).limit(limit).sort(options.sort || { createdAt: -1 }),
    Model.countDocuments(query)
  ]);
  
  return {
    data,
    pagination: {
      current: page,
      pages: Math.ceil(total / limit),
      total,
      hasNext: page < Math.ceil(total / limit),
      hasPrev: page > 1
    }
  };
};
```

### Search Helper
```javascript
const buildSearchQuery = (searchTerm, fields) => {
  if (!searchTerm) return {};
  
  return {
    $or: fields.map(field => ({
      [field]: { $regex: searchTerm, $options: 'i' }
    }))
  };
};

// Usage
const query = buildSearchQuery(req.query.search, ['name', 'email', 'profile.firstName']);
const users = await User.find(query);
```

### Audit Trail Pattern
```javascript
const auditSchema = new mongoose.Schema({
  action: { type: String, required: true }, // 'create', 'update', 'delete'
  collection: { type: String, required: true },
  documentId: { type: mongoose.Schema.Types.ObjectId, required: true },
  changes: mongoose.Schema.Types.Mixed,
  userId: { type: mongoose.Schema.Types.ObjectId, ref: 'User' },
  timestamp: { type: Date, default: Date.now }
});

// Middleware to create audit logs
userSchema.post('save', async function() {
  if (this.isNew) {
    await AuditLog.create({
      action: 'create',
      collection: 'User',
      documentId: this._id,
      changes: this.toObject()
    });
  }
});
```

## 🚀 Quick Reference Commands

```javascript
// CREATE
Model.create(data)
Model.insertMany([data1, data2])
new Model(data).save()

// READ
Model.find(query)
Model.findById(id)
Model.findOne(query)
Model.countDocuments(query)
Model.aggregate(pipeline)

// UPDATE
Model.updateOne(filter, update)
Model.updateMany(filter, update)
Model.findByIdAndUpdate(id, update, options)
Model.findOneAndUpdate(filter, update, options)

// DELETE
Model.deleteOne(filter)
Model.deleteMany(filter)
Model.findByIdAndDelete(id)
Model.findOneAndDelete(filter)

// OPTIONS
{ new: true } // Return updated document
{ upsert: true } // Create if not exists
{ runValidators: true } // Run schema validators
{ lean: true } // Return plain objects
{ session } // Use transaction
```

---
*Last updated: August 2025*